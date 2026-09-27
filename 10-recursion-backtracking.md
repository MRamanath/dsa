# Recursion & Backtracking

## Learning Objectives

By the end of this topic, you should understand:

- how a base case stops recursion and a recursive case reduces the remaining work
- how to name a shrinking measure and use it to prove termination
- how call frames accumulate, return values flow back, and unwind-time work differs from descent-time work
- how recursion trees reveal repeated work, branching, depth, and output-sensitive cost
- when overlapping subproblems justify memoization
- how backtracking follows choose, explore, and unchoose while restoring state on every exit path
- how duplicate choices, pruning, and copying decisions affect correctness and complexity
- how JavaScript's finite call stack constrains recursive APIs
- how Unicode code points differ from UTF-16 code units and user-perceived grapheme clusters
- how to expose bounded recursive traces safely from a NestJS endpoint

## Prerequisites

- functions, parameters, return values, arrays, loops, and conditionals
- Big-O notation and the distinction between auxiliary and output space
- TypeScript strict mode, `bigint`, `Number.isSafeInteger`, and `RangeError`
- strings, UTF-16, spread iteration, and basic NestJS request/response concepts

## Core Theory

### Base case and recursive case

A recursive function solves a problem by calling itself on a smaller instance. Every correct design needs both parts:

- The **base case** returns directly for the smallest valid state.
- The **recursive case** reduces the state, delegates the smaller problem, and combines its result.

Without a reachable base case, calls continue until JavaScript throws a call-stack overflow. A base case that exists but cannot be reached is equally broken.

### Shrinking measure and termination proof

Do not justify termination with only "the input gets smaller." Name a measure that is a nonnegative integer:

- counting down: `current`
- moving indexes inward: `right - left + 1`
- consuming digits: the number of decimal digits
- searching choices: the number of undecided positions

Then prove two facts: every recursive edge strictly decreases that measure, and the base case handles its minimum. For `factorial(n)`, the measure is `n`; `n - 1 < n`, and `n = 0` returns directly. Validation must happen before recursion so invalid values cannot bypass the proof.

### Call stack, return, and unwind

Each call owns a frame containing its parameters, local variables, and return location. Calls made during **descent** push frames. Reaching the base case starts **unwinding**: frames return in last-in, first-out order.

```text
factorial(3)
  waits for factorial(2)
    waits for factorial(1)
      waits for factorial(0)
      returns 1n
    returns 1n * 1n
  returns 2n * 1n
returns 3n * 2n
```

Work before the recursive call follows descent order. Work after it follows reverse, unwind order. Problem 131 uses that distinction to recurse down first and append numbers while returning.

### Finite JavaScript stacks

JavaScript engines have finite call stacks, and the language does not guarantee tail-call optimization in common runtimes. Maximum safe depth varies by engine, flags, frame size, and surrounding application code. Recursive teaching examples in this chapter therefore use bounded contracts where depth could otherwise grow without limit. A bound such as 1,000 is an educational risk control, not a universal proof that every runtime can support exactly that many frames.

### Recursion trees

A recursion tree draws one node per call and one edge per recursive subcall. A one-branch recurrence such as factorial has linear depth and call count. Naive Fibonacci branches twice:

```text
fib(5)
+-- fib(4)
|   +-- fib(3)
|   `-- fib(2)
`-- fib(3)          repeated state
    +-- fib(2)      repeated state
    `-- fib(1)
```

The same states appear in multiple branches. The tree grows exponentially even though there are only `n + 1` distinct inputs from `0` through `n`.

### Overlapping subproblems and memoization

When different branches solve the same state, cache the result by a complete state key. Top-down memoization preserves the recursive definition while ensuring each distinct state is computed once. Memoization helps only when states overlap; caching a traversal in which every state is unique adds storage without avoiding work.

The key must describe everything that affects the answer. A backtracking state may require an index plus remaining capacity, not merely the index. Mutable objects are dangerous keys unless their identity is intentionally the state.

### Backtracking: choose, explore, unchoose

Backtracking recursively searches a decision tree:

```text
for each legal choice:
    choose: mutate path/state
    explore: recurse on the next decision
    unchoose: restore exactly what choose changed
```

State restoration is required after success, ordinary failure, early stopping, and exceptions. If exploration can throw, place restoration in `finally`; if it returns a boolean, restore before returning that boolean. Shared mutable state entering a sibling branch must be identical to the state that existed before the previous sibling's choice.

### Pruning and duplicate choices

**Pruning** skips a branch only when a proven condition says it cannot produce a required answer. Examples include exceeding a target of nonnegative values or lacking enough remaining positions to complete a fixed-length result. A heuristic that merely looks unpromising is not a correctness-preserving prune.

Duplicate input choices can generate duplicate outputs. A common pattern sorts choices, then at one recursion depth skips an equal value after its first occurrence. The depth condition matters: equal values may still be valid at different positions in the same result. Sorting changes order and usually requires copying when the caller's input must remain unchanged.

### Mutation, copying, and ownership

Local mutation is often efficient: push a choice, recurse, then pop it. When recording a completed path, copy it with an operation such as `results.push([...path])`; otherwise every result may reference the same later-mutated array. Public contracts should state whether inputs can be changed. The solutions here do not mutate caller-owned inputs and return fresh arrays or strings where applicable.

Validation should be atomic: reject the complete invalid scalar or sequence before recursive work or externally visible mutation begins. Local helper state can then rely on established invariants.

### Unicode code points

JavaScript string indexes address UTF-16 code units, so an emoji may occupy two indexed units. `[...text]` iterates Unicode code points and is the contract used by Problems 135 and 136. Code points still are not grapheme clusters: a family emoji joined by zero-width joiners and a letter plus a combining mark can contain multiple code points. These problems perform no normalization, locale folding, or grapheme segmentation.

### Production recursion and bounded traces

An educational NestJS endpoint should call only the intended optimized recursive solution. Validate DTO shape and algorithm contracts before recursion, map `RangeError` to HTTP `400`, and cap trace events, for example at 200. Return a `truncated` flag so omitted events are explicit.

Trace records should contain compact state such as depth, arguments, phase, and a bounded result preview. Never serialize whole recursion trees, memo tables, or growing output snapshots per call. Such tracing can turn linear work into quadratic copying and produce oversized responses. `bigint` values are not directly JSON-serializable, so endpoints must encode them as decimal strings. Production services should prefer an iterative equivalent, queue, or explicit stack when user-controlled depth can exceed a conservative bound.

## Visual Mental Model

Think of each recursive call as a card placed on a stack. The card records what must happen after the smaller call returns.

```text
DESCENT                         UNWIND

call(3)                         call(3) finishes
  call(2)                       call(2) finishes
    call(1)                     call(1) finishes
      base() returns      ->    base result rises upward

frames grow downward            frames leave upward
```

Backtracking adds a reversible choice around each descent:

```text
state [A]
  choose B -> [A, B]
    explore
  unchoose B -> [A]
  choose C -> [A, C]     sibling sees restored [A]
```

A recursion tree visualizes calls, while the runtime call stack contains only the currently active root-to-leaf path. Tree width can be exponential even when stack depth is linear.

## Complexity Concepts

### Call count and stack depth

Time often follows the number of calls plus the work done in each call. Auxiliary stack space follows maximum active depth, not total call count. Naive Fibonacci makes exponentially many calls but keeps only `O(n)` frames active at once.

### Recurrences

A recurrence describes work in terms of smaller inputs:

- `T(n) = T(n - 1) + O(1)` gives `O(n)` work.
- `T(n) = T(n - 1) + T(n - 2) + O(1)` gives exponential naive Fibonacci work.
- Memoized Fibonacci computes `O(n)` states, each once in the operation-count model.

### Output-sensitive complexity

When an algorithm must return many answers, time cannot be smaller than the data it emits. Generating `k` results of average length `m` already requires `Omega(km)` output work and space. Backtracking complexity should name both the search-tree bound and the required output size instead of calling all exponential behavior avoidable.

### Arithmetic is not always constant time

Operation-count analysis may count additions or multiplications, but `bigint` operands grow. Fibonacci addition and factorial multiplication therefore become more expensive with `n`. Factorial has `Theta(n log n)` result bits by a broad asymptotic bound, so merely representing or returning the exact answer is not constant-space output.

### Auxiliary space versus output space

Report recursive frames, memo tables, and local arrays as auxiliary space. Report the required returned array, string, or exact integer representation separately as output space. If a function builds an output array while recursing, both the call stack and output coexist even though only one is temporary.

### Copying costs

String concatenation inside each recursive frame can repeatedly copy growing prefixes and become quadratic. Prefer a single output array followed by one `join`. In backtracking, copying a path only when emitting a result usually avoids copying at every internal node.

## Common Patterns

### Countdown, then unwind

```text
solve(current):
    if current is base: return
    solve(current - 1)
    process current during unwind
```

### Divide a scalar state

```text
digits(value):
    if value < 10: return value
    return value mod 10 + digits(floor(value / 10))
```

### Two indexes moving inward

```text
solve(left, right):
    if left >= right: return base result
    process endpoints
    return solve(left + 1, right - 1)
```

### Top-down memoization

```text
solve(state):
    if base: return base value
    if memo has state: return cached value
    answer = combine recursive subproblems
    memo[state] = answer
    return answer
```

### Choose, explore, unchoose

```text
search(state, path):
    if complete: emit a copy of path; return
    for each legal nonduplicate choice:
        choose
        try:
            search(next state, path)
        finally:
            unchoose
```

# Problems

## Problem 131 — Print Numbers using Recursion

### 1. Problem Statement
Given an integer `n` from `0` through `1000`, return a fresh `number[]` containing every integer from `1` through `n`. For `n = 0`, return `[]`. Despite the title, the API has no printing or logging side effects; it returns a sequence that a caller may print. Validate `n` before allocating output or recursing.

### 2. Example
```text
Input:  n = 5
Output: [1, 2, 3, 4, 5]

Input:  n = 0
Output: []
```

### 3. Understand the Problem
The order is ascending, but the recursive helper counts downward. It reaches zero first, then pushes each saved `current` value while frames unwind. The upper bound makes the teaching contract explicit, though JavaScript stack limits remain runtime-dependent.

#### In Simple Terms
Walk down to zero, then collect the waiting numbers as the calls return.

### 4. Real-World Analogy
Imagine placing numbered cards `n` through `1` on a pile. Removing the cards reveals `1` first, followed by `2`, so the unwind order is ascending.

### 5. Key Observation
Code after a recursive call runs during unwind. Calling with `current - 1` before pushing `current` reverses a descending call order into ascending output.

### 6. Pattern
```text
Pattern: Countdown Then Process on Unwind
Why this pattern applies: desired output is the reverse of recursive descent
How to recognize it: smaller values must appear before the current value
Similar problems: recursive range construction, postorder traversal, stack reversal
```

### 7. Brute Force
After validating the entire scalar contract, iterate from `1` through `n` and append each number. This direct baseline avoids recursion and is the production-friendly option for larger valid bounds.

#### Pseudocode
```text
validate n is an integer from 0 through 1000
result = empty array
for value from 1 through n: append value
return result
```

#### Brute-Force TypeScript
```typescript
function printNumbersIterative(n: number): number[] {
  if (!Number.isInteger(n) || n < 0 || n > 1000) {
    throw new RangeError("n must be an integer from 0 through 1000");
  }

  const result: number[] = [];
  for (let value = 1; value <= n; value += 1) {
    result.push(value);
  }
  return result;
}
```

#### Visualization
```text
n = 5
[] -> [1] -> [1, 2] -> [1, 2, 3] -> [1, 2, 3, 4] -> [1, 2, 3, 4, 5]
```

#### Dry Run
For `n = 3`, validation succeeds. The loop appends `1`, then `2`, then `3`, and returns `[1, 2, 3]`. For `n = 0`, the loop does not run and the fresh empty array is returned.

#### Brute-Force Complexity
Validation is `O(1)`. The loop takes `O(n)` time. The returned array requires `O(n)` output space, while auxiliary space excluding output is `O(1)`.

### 8. Why Brute Force Is Not Ideal
The iterative baseline is actually safer on the JavaScript call stack and has the same asymptotic time. It is called brute force only to preserve the chapter comparison; recursion is pedagogical here, not an asymptotic improvement.

### 9. Optimal Approach
Validate before work. Create one result array, then call `collect(n)`. The helper returns at zero; otherwise it first calls `collect(current - 1)` and then pushes `current`. No intermediate arrays are copied.

### 10. Visualization
```text
collect(3)
  collect(2)
    collect(1)
      collect(0) -> base
    push 1 -> [1]
  push 2   -> [1, 2]
push 3     -> [1, 2, 3]
```

### 11. Step-by-Step Dry Run
| Phase | Active `current` | Depth | Result |
|---|---:|---:|---|
| descend | 3 | 1 | `[]` |
| descend | 2 | 2 | `[]` |
| descend | 1 | 3 | `[]` |
| base | 0 | 4 | `[]` |
| unwind | 1 | 3 | `[1]` |
| unwind | 2 | 2 | `[1, 2]` |
| unwind | 3 | 1 | `[1, 2, 3]` |

### 12. TypeScript Solution
```typescript
function printNumbersRecursive(n: number): number[] {
  if (!Number.isInteger(n) || n < 0 || n > 1000) {
    throw new RangeError("n must be an integer from 0 through 1000");
  }

  const result: number[] = [];
  const collect = (current: number): void => {
    if (current === 0) return;
    collect(current - 1);
    result.push(current);
  };

  collect(n);
  return result;
}
```

### 13. Code Walkthrough
The public function rejects invalid input before creating the output. `current` is the shrinking measure and decreases by one until the zero base case. Each frame then appends its own value during unwind, so one shared local array receives ascending numbers without printing or repeated copying.

### 14. Complexity Analysis
There are `n + 1` helper calls and `n` pushes, so time is `O(n)`. The call stack uses `O(n)` auxiliary space, and the returned array uses `O(n)` output space. The contract bounds depth at 1,000, but an iterative implementation remains preferable when stack guarantees matter.

### 15. Edge Cases
- `n = 0` returns a fresh `[]`.
- `n = 1` returns `[1]`.
- `n = 1000` is the largest accepted teaching input.
- Negative, fractional, `NaN`, infinite, and values above `1000` throw `RangeError` before output allocation.

### 16. Common Mistakes
- Printing with `console.log` instead of returning the sequence.
- Pushing before recursion and producing descending order.
- Omitting the zero base case.
- Validating inside the helper after output has already changed.
- Claiming `O(1)` auxiliary space while ignoring recursive frames.

### 17. Interview Follow-ups
1. Return numbers in descending order. 2. Return only even numbers. 3. Support an inclusive `[start, end]` range. 4. Replace recursion with an explicit stack.

### 18. Variations
- Recursive countdown. - Recursive array fill. - Postorder tree output. - Lazy generator. - Chunked iteration for large bounds.

### 19. Pattern Connections
This is the simplest example of postorder-style work: recurse first and process the current state second. The call stack temporarily owns the values that will later be emitted, just as a depth-first traversal remembers parent nodes while visiting children.

### 20. NestJS Project Connection
Expose `POST /api/dsa/recursion/print-numbers` with body `{ n: number }`. Validate the DTO and function contract before calling only `printNumbersRecursive`; map `RangeError` to HTTP `400`. Return `{ numbers, trace, truncated }` with at most 200 compact optimized events such as `{ phase, current, depth }`. Do not snapshot the growing array in each event, and document that recursion depth is bounded but runtime stack capacity is not guaranteed.

### 21. Mini Challenge
Return `[1, 2, ..., n]` recursively without mutating an output array. Analyze why repeatedly constructing arrays with spread can make the implementation quadratic in copying work.

## Problem 132 — Factorial

### 1. Problem Statement
Given an integer `n` from `0` through `1000`, return the exact `bigint` factorial `n!`. Define `0! = 1n`. Validate before multiplication or recursion. The result must never be converted through `number`.

### 2. Example
```text
Input:  n = 5
Output: 120n

Input:  n = 0
Output: 1n
```

### 3. Understand the Problem
Factorial multiplies every positive integer through `n`. The recursive identity is `n! = n * (n - 1)!`, with `0! = 1`. Exactness requires `bigint` because factorial quickly exceeds the safe-integer range.

#### In Simple Terms
Multiply `n` by the exact factorial of the number immediately below it until zero supplies the multiplicative identity.

### 4. Real-World Analogy
Counting arrangements is like filling `n` labeled positions: there are `n` choices for the first, then `n - 1`, and so on, so the counts multiply.

### 5. Key Observation
The mathematical definition already has one smaller subproblem. The shrinking measure is `n`, and converting each validated integer factor with `BigInt` preserves exact arithmetic.

### 6. Pattern
```text
Pattern: Linear Recursive Reduction
Why this pattern applies: n! combines n with exactly one smaller factorial
How to recognize it: definition has one base value and one n - 1 recurrence
Similar problems: products of ranges, recursive powers, multiplicative folds
```

### 7. Brute Force
Validate first, initialize the exact accumulator to `1n`, and multiply by each integer from `2` through `n`. This iterative baseline performs the same `n - 1` meaningful multiplications without recursive frames.

#### Pseudocode
```text
validate n is an integer from 0 through 1000
result = 1n
for factor from 2 through n: result = result * BigInt(factor)
return result
```

#### Brute-Force TypeScript
```typescript
function factorialIterative(n: number): bigint {
  if (!Number.isInteger(n) || n < 0 || n > 1000) {
    throw new RangeError("n must be an integer from 0 through 1000");
  }

  let result = 1n;
  for (let factor = 2; factor <= n; factor += 1) {
    result *= BigInt(factor);
  }
  return result;
}
```

#### Visualization
```text
5! = 1n * 2n * 3n * 4n * 5n
   = 120n
```

#### Dry Run
For `n = 4`, the accumulator starts at `1n`, then becomes `2n`, `6n`, and `24n`. For `n = 0`, the loop is skipped and the identity `1n` is returned.

#### Brute-Force Complexity
The operation-count model has `O(n)` `bigint` multiplications and `O(1)` scalar auxiliary state. Arithmetic is not constant time: operands grow with the result. The exact output itself has `Theta(n log n)` bits by a broad asymptotic bound, and the accumulator occupies comparable storage.

### 8. Why Brute Force Is Not Ideal
It is ideal for stack safety and is not asymptotically worse in multiplication count. The recursive form is pedagogical, mirrors the definition, and is not asymptotically superior; production code should usually keep the loop.

### 9. Optimal Approach
After complete validation, define `compute(current)`. Return `1n` at zero. Otherwise return `BigInt(current) * compute(current - 1)`. The recursive call reaches the base before multiplication occurs during unwind.

### 10. Visualization
```text
compute(4)
= 4n * compute(3)
= 4n * 3n * compute(2)
= 4n * 3n * 2n * compute(1)
= 4n * 3n * 2n * 1n * compute(0)
= 24n
```

### 11. Step-by-Step Dry Run
| Phase | `current` | Depth | Returned/held value |
|---|---:|---:|---|
| descend | 4 | 1 | waits for `compute(3)` |
| descend | 3 | 2 | waits for `compute(2)` |
| descend | 2 | 3 | waits for `compute(1)` |
| descend | 1 | 4 | waits for `compute(0)` |
| base | 0 | 5 | `1n` |
| unwind | 1 | 4 | `1n` |
| unwind | 2 | 3 | `2n` |
| unwind | 3 | 2 | `6n` |
| unwind | 4 | 1 | `24n` |

### 12. TypeScript Solution
```typescript
function factorialRecursive(n: number): bigint {
  if (!Number.isInteger(n) || n < 0 || n > 1000) {
    throw new RangeError("n must be an integer from 0 through 1000");
  }

  const compute = (current: number): bigint => {
    if (current === 0) return 1n;
    return BigInt(current) * compute(current - 1);
  };

  return compute(n);
}
```

### 13. Code Walkthrough
Validation establishes a finite nonnegative integer before the helper starts. Each call reduces `current` by one, proving termination at zero. Multiplication uses only `bigint` operands, so no unsafe `number` result is introduced, and unwind combines one factor with the exact smaller factorial.

### 14. Complexity Analysis
In an operation-count model, the function performs `O(n)` multiplications and uses `O(n)` call-stack frames. That model does not make multiplication constant time: operand bit lengths grow throughout the unwind. The returned exact value has `Theta(n log n)` bits by a broad bound, so output representation alone grows with `n`; peak arithmetic storage also includes growing intermediate `bigint` values.

### 15. Edge Cases
- `0` and `1` return `1n`.
- `1000` returns an exact large `bigint`.
- Negative, fractional, nonfinite, and values above `1000` throw before recursion.
- The result is a `bigint`, not a JSON-ready `number`.

### 16. Common Mistakes
- Returning `1` instead of `1n` and mixing numeric domains.
- Computing with `number` and converting the already-rounded result to `bigint`.
- Using `n = 1` as the only base case and failing for zero.
- Calling each multiplication `O(1)` despite growing operands.
- Claiming recursion improves asymptotic multiplication count.

### 17. Interview Follow-ups
1. Count trailing zeros without constructing `n!`. 2. Compute factorial modulo a positive modulus. 3. Compare balanced product trees. 4. Estimate decimal digit count.

### 18. Variations
- Double factorial. - Falling factorial. - Binomial coefficients. - Product of an inclusive range. - Prime-factor representation.

### 19. Pattern Connections
Factorial is a multiplicative fold over a range and a one-child recursion tree. It demonstrates that mathematical elegance, stack safety, and bit complexity are separate concerns.

### 20. NestJS Project Connection
Expose `POST /api/dsa/recursion/factorial` with `{ n: number }`. Call only `factorialRecursive` after DTO validation, map `RangeError` to HTTP `400`, and encode the answer as `{ factorial: result.toString() }` because JSON cannot serialize `bigint`. Cap optimized trace events such as `{ phase, current, depth, partialDecimal }` at 200; bound or omit partial values to prevent response amplification, and surface `truncated` plus the recursion-depth limit.

### 21. Mini Challenge
Implement a recursive balanced product range for `1..n`, then compare its recursion depth and `bigint` multiplication shapes with the linear recurrence without claiming constant-time arithmetic.

## Problem 133 — Fibonacci

### 1. Problem Statement
Given an integer `n` from `0` through `1000`, return the exact Fibonacci value `F(n)` as a `bigint`, where `F(0) = 0n`, `F(1) = 1n`, and `F(n) = F(n - 1) + F(n - 2)`. Validate before recursion.

### 2. Example
```text
Input:  n = 10
Output: 55n

Input:  n = 0
Output: 0n
```

### 3. Understand the Problem
The recurrence branches into two smaller indexes. Naive recursion recomputes the same values many times. Top-down memoization stores each exact result so future calls for that index return immediately.

#### In Simple Terms
Remember every Fibonacci value after computing it once so repeated branches can reuse it.

### 4. Real-World Analogy
Without notes, two coworkers repeatedly recalculate the same subtotals. A shared lookup table lets the first calculation answer every later request for that subtotal.

### 5. Key Observation
The naive recursion tree has exponentially many call occurrences but only `n + 1` distinct index states. The index alone completely identifies a subproblem, so it is a valid memo key.

### 6. Pattern
```text
Pattern: Top-Down Dynamic Programming
Why this pattern applies: recursive branches overlap on the same indexes
How to recognize it: a recursion tree repeats identical state labels
Similar problems: climbing stairs, grid paths, minimum-cost recurrences
```

### 7. Brute Force
Validate once, then translate the recurrence directly. This is intentionally naive and suitable only for tiny `n`; although its contract accepts through `1000`, large values require infeasible time long before the bound and may also encounter runtime stack constraints.

#### Pseudocode
```text
validate n is an integer from 0 through 1000
fib(index):
    if index < 2: return BigInt(index)
    return fib(index - 1) + fib(index - 2)
return fib(n)
```

#### Brute-Force TypeScript
```typescript
function fibonacciNaive(n: number): bigint {
  if (!Number.isInteger(n) || n < 0 || n > 1000) {
    throw new RangeError("n must be an integer from 0 through 1000");
  }

  const compute = (index: number): bigint => {
    if (index < 2) return BigInt(index);
    return compute(index - 1) + compute(index - 2);
  };

  return compute(n);
}
```

#### Visualization
```text
fib(5)
+-- fib(4)
|   +-- fib(3)
|   `-- fib(2)
`-- fib(3)  <- repeated subtree
```

#### Dry Run
For `n = 5`, the root requests `fib(4)` and `fib(3)`. Those branches independently request `fib(3)`, `fib(2)`, and smaller states many times before combining base values into `5n`.

#### Brute-Force Complexity
The call count is `O(phi^n)`, where `phi` is the golden ratio, and maximum stack depth is `O(n)`. Exact `bigint` additions have growing bit cost, so the call-count bound is an operation-count simplification rather than a full bit-complexity claim.

### 8. Why Brute Force Is Not Ideal
It discards completed subproblems and rebuilds entire repeated subtrees. The exponential call count makes even moderate `n` impractical despite there being only linear many distinct states.

### 9. Optimal Approach
Validate before creating recursive state. Seed a memo with `0 -> 0n` and `1 -> 1n`. For any other index, return a cached value when present; otherwise compute the two smaller states, store their exact sum, and return it.

### 10. Visualization
```text
compute(5)
  compute(4)
    compute(3)
      compute(2) -> store 1n
      compute(1) -> cached 1n
    store 2n
    compute(2) -> cached 1n
  store 3n
  compute(3) -> cached 2n
store 5n
```

### 11. Step-by-Step Dry Run
| Event | Index | Depth | Memo after event |
|---|---:|---:|---|
| seed | - | 0 | `{0: 0n, 1: 1n}` |
| store | 2 | 4 | `..., 2: 1n` |
| store | 3 | 3 | `..., 3: 2n` |
| hit | 2 | 3 | unchanged |
| store | 4 | 2 | `..., 4: 3n` |
| hit | 3 | 2 | unchanged |
| store | 5 | 1 | `..., 5: 5n` |

### 12. TypeScript Solution
```typescript
function fibonacciMemoized(n: number): bigint {
  if (!Number.isInteger(n) || n < 0 || n > 1000) {
    throw new RangeError("n must be an integer from 0 through 1000");
  }

  const memo = new Map<number, bigint>([
    [0, 0n],
    [1, 1n],
  ]);

  const compute = (index: number): bigint => {
    const cached = memo.get(index);
    if (cached !== undefined) return cached;

    const value = compute(index - 1) + compute(index - 2);
    memo.set(index, value);
    return value;
  };

  return compute(n);
}
```

### 13. Code Walkthrough
The full range check occurs before recursion. Seeded base values terminate every branch. Because Fibonacci values are nonnegative and the memo stores `bigint`, `undefined` unambiguously means a miss. The first descent computes each missing index; later overlapping calls are constant-time map lookups in the operation-count model.

### 14. Complexity Analysis
There are `O(n)` distinct states and each is computed once, so the operation-count model has `O(n)` additions and expected `O(n)` map activity. The memo and maximum call stack each use `O(n)` entries/frames. Exact addition cost grows with Fibonacci bit length, which is `Theta(n)` for `F(n)`; the returned `bigint` also requires `Theta(n)` bits.

### 15. Edge Cases
- `n = 0` returns `0n`; `n = 1` returns `1n`.
- `n = 2` returns `1n`, exercising both seeds.
- `n = 1000` remains exact.
- Negative, fractional, nonfinite, and oversized values throw before memo creation.

### 16. Common Mistakes
- Memoizing after both repeated branches have already been recomputed.
- Using `number` and losing exactness for larger indexes.
- Testing cache presence by truthiness, which fails for cached `0n`.
- Calling the naive version `O(2^n)` without noting `O(phi^n)` is a tighter call bound.
- Ignoring growing `bigint` addition cost.

### 17. Interview Follow-ups
1. Implement bottom-up iteration with `O(1)` state variables. 2. Return all values through `F(n)`. 3. Use fast doubling. 4. Compute modulo a positive modulus.

### 18. Variations
- Climbing stairs. - Tribonacci. - Linear recurrences. - Fast doubling. - Matrix exponentiation. - Bounded memo caches.

### 19. Pattern Connections
Fibonacci is the canonical bridge from recursion trees to dynamic programming. Memoization collapses repeated call occurrences into a directed acyclic graph of distinct states, trading `O(n)` storage for an exponential reduction in operation count.

### 20. NestJS Project Connection
Expose `POST /api/dsa/recursion/fibonacci` with `{ n: number }`. Invoke only `fibonacciMemoized`, convert the result and any traced values to decimal strings, and map invalid input to HTTP `400`. Return a maximum of 200 optimized events such as `{ index, depth, action: "hit" | "store", value }` plus `truncated`; never serialize the full memo per event. The 1,000 bound limits teaching depth, but an iterative or fast-doubling service is safer for broader production contracts.

### 21. Mini Challenge
Implement fast doubling recursively using `F(2k)` and `F(2k + 1)` identities, preserving exact `bigint` arithmetic and reducing recursion depth to `O(log n)`.

## Problem 134 — Sum of Digits

### 1. Problem Statement
Given any safe integer, including a negative safe integer, return the sum of the decimal digits of its absolute value. Return `0` for input `0`. Validate before conversion or recursion. `Math.abs(Number.MIN_SAFE_INTEGER)` is `Number.MAX_SAFE_INTEGER` and remains safe.

### 2. Example
```text
Input:  -5029
Output: 16

Input:  0
Output: 0
```

### 3. Understand the Problem
The sign does not contribute a digit. After validation, take the absolute value once. Arithmetic recursion removes the last digit with floor division by ten and adds that digit with remainder.

#### In Simple Terms
Peel off the last decimal digit, add it, and repeat on the digits to its left.

### 4. Real-World Analogy
It is like reading a stack of digit tiles from right to left: record the visible tile, remove it, and continue until one tile remains.

### 5. Key Observation
For a nonnegative integer, `value % 10` is the last digit and `Math.floor(value / 10)` removes it. The decimal digit count strictly decreases whenever `value >= 10`.

### 6. Pattern
```text
Pattern: Recursive Numeric Decomposition
Why this pattern applies: division separates one digit from a smaller prefix
How to recognize it: the base and recurrence follow decimal place value
Similar problems: digit count, digital root, integer reversal
```

### 7. Brute Force
Validate the safe integer, take its absolute value, convert it to a decimal string, and sum each character's numeric value. The conversion allocates `O(d)` string space for `d` digits.

#### Pseudocode
```text
validate input is a safe integer
digits = decimal string of absolute input
sum = 0
for each character in digits: add its numeric value
return sum
```

#### Brute-Force TypeScript
```typescript
function sumOfDigitsString(input: number): number {
  if (!Number.isSafeInteger(input)) {
    throw new RangeError("input must be a safe integer");
  }

  const digits = Math.abs(input).toString();
  let sum = 0;
  for (const digit of digits) {
    sum += Number(digit);
  }
  return sum;
}
```

#### Visualization
```text
-5029 -> "5029" -> 5 + 0 + 2 + 9 -> 16
```

#### Dry Run
Validation accepts `-5029`. Its absolute decimal string is `"5029"`; the running sum changes from `0` to `5`, stays `5`, then becomes `7` and finally `16`.

#### Brute-Force Complexity
For `d` decimal digits, conversion and scanning take `O(d)` time. The generated decimal string uses `O(d)` auxiliary space; the numeric result uses `O(1)` output space.

### 8. Why Brute Force Is Not Ideal
The string baseline is clear and asymptotically linear, but it changes representation and allocates a digit string. Arithmetic recursion demonstrates numeric decomposition, though it trades the string for call-stack frames rather than improving the asymptotic bound.

### 9. Optimal Approach
After validation, compute `value = Math.abs(input)`. The helper returns `value` when it is below ten, including zero. Otherwise return `value % 10 + helper(Math.floor(value / 10))`.

### 10. Visualization
```text
helper(5029)
= 9 + helper(502)
= 9 + 2 + helper(50)
= 9 + 2 + 0 + helper(5)
= 9 + 2 + 0 + 5
= 16
```

### 11. Step-by-Step Dry Run
| Phase | Value | Last digit | Remaining prefix | Depth |
|---|---:|---:|---:|---:|
| descend | 5029 | 9 | 502 | 1 |
| descend | 502 | 2 | 50 | 2 |
| descend | 50 | 0 | 5 | 3 |
| base | 5 | 5 | - | 4 |
| unwind | 50 | 0 | partial `5` | 3 |
| unwind | 502 | 2 | partial `7` | 2 |
| unwind | 5029 | 9 | result `16` | 1 |

### 12. TypeScript Solution
```typescript
function sumOfDigitsRecursive(input: number): number {
  if (!Number.isSafeInteger(input)) {
    throw new RangeError("input must be a safe integer");
  }

  const helper = (value: number): number => {
    if (value < 10) return value;
    return (value % 10) + helper(Math.floor(value / 10));
  };

  return helper(Math.abs(input));
}
```

### 13. Code Walkthrough
The public function accepts every safe integer and rejects all other numeric values before work. Taking the absolute value is safe even for `Number.MIN_SAFE_INTEGER`, whose magnitude equals `Number.MAX_SAFE_INTEGER`. Each helper call removes one decimal digit, and the bounded safe-integer range keeps depth small.

### 14. Complexity Analysis
For `d` decimal digits, the helper makes `d` calls, giving `O(d)` arithmetic operations and `O(d)` call-stack space. The returned sum uses `O(1)` output space. Since safe integers have a fixed maximum decimal length in JavaScript, `d <= 16`, but expressing complexity in `d` describes the algorithmic pattern.

### 15. Edge Cases
- `0` returns `0` from the base case.
- Negative values ignore the sign.
- Internal zeros, as in `1002`, contribute zero without ending recursion early.
- `Number.MIN_SAFE_INTEGER` is valid and remains safe after `Math.abs`.
- `NaN`, infinities, fractions, and unsafe integers throw `RangeError`.

### 16. Common Mistakes
- Treating a negative remainder as a digit instead of taking absolute value once.
- Using `parseInt` on slices when arithmetic already defines the reduction.
- Returning at any zero digit rather than only when the whole remaining value is below ten.
- Accepting integers outside the safe range.
- Claiming the recursive version uses `O(1)` auxiliary space.

### 17. Interview Follow-ups
1. Count digits. 2. Compute a digital root. 3. Support an arbitrary radix. 4. Accept an exact integer decimal string beyond the safe range.

### 18. Variations
- Product of digits. - Number reversal. - Palindromic integer. - Digit-frequency counts. - Repeated digit sum.

### 19. Pattern Connections
Numeric decomposition is the scalar counterpart of shrinking an array slice. The quotient is the smaller subproblem, the remainder is local work, and the digit count supplies a direct termination measure.

### 20. NestJS Project Connection
Expose `POST /api/dsa/recursion/sum-of-digits` with `{ input: number }`. Validate JSON shape and call only `sumOfDigitsRecursive`, mapping `RangeError` to HTTP `400`. Return `{ sum, trace, truncated }` with no more than 200 optimized events such as `{ value, lastDigit, depth, phase }`; never trace the baseline string conversion. Safe-integer validation also keeps recursive depth naturally bounded.

### 21. Mini Challenge
Generalize the recursive helper to a validated radix from `2` through `36`, and state how the digit count and maximum recursion depth change with that radix.

## Problem 135 — Reverse String using Recursion

### 1. Problem Statement
Given a string containing at most 1,000 Unicode code points, return a new string with its code-point sequence reversed. Perform no Unicode normalization. Build `[...text]`, then throw `RangeError` if its length exceeds `1000`, before recursion or output mutation. The empty string returns `""`.

### 2. Example
```text
Input:  "A😀B"
Output: "B😀A"

Input:  ""
Output: ""
```

### 3. Understand the Problem
String indexing addresses UTF-16 code units and can split a supplementary code point. Spreading produces whole code-point strings. The recursive helper moves two indexes inward and writes mirrored entries into one output array, avoiding repeated string concatenation.

#### In Simple Terms
Turn the text into whole code points, place the outside pair in opposite output positions, and recurse inward.

### 4. Real-World Analogy
Two people stand at opposite ends of a row of tiles and copy each other's tile into the mirrored position, then both step toward the center.

### 5. Key Observation
The code point at input index `right` belongs at output index `left`, and the code point at `left` belongs at `right`. One recursive step settles both outer positions.

### 6. Pattern
```text
Pattern: Two Recursive Indexes Moving Inward
Why this pattern applies: mirrored endpoints can be settled as a pair
How to recognize it: output position i depends on input position n - 1 - i
Similar problems: palindrome checks, in-place reversal, symmetric comparisons
```

### 7. Brute Force
Convert the string to a code-point array, enforce the 1,000-code-point limit immediately, then use array reversal and one join. This concise baseline is usually preferable in production.

#### Pseudocode
```text
codePoints = spread text into Unicode code points
if codePoints length exceeds 1000: throw RangeError
reverse codePoints
join and return
```

#### Brute-Force TypeScript
```typescript
function reverseStringBuiltIn(text: string): string {
  const codePoints = [...text];
  if (codePoints.length > 1000) {
    throw new RangeError("text must contain at most 1000 Unicode code points");
  }
  return codePoints.reverse().join("");
}
```

#### Visualization
```text
["A", "😀", "B"] -> reverse -> ["B", "😀", "A"] -> "B😀A"
```

#### Dry Run
For `"A😀B"`, spread creates three entries rather than four UTF-16 code units. The limit passes, reverse swaps the outer entries, and join returns `"B😀A"`.

#### Brute-Force Complexity
For `n` Unicode code points, spreading, reversing, and joining take `O(n)` time. The code-point array and returned string each use `O(n)` space; the array is auxiliary and the string is required output.

### 8. Why Brute Force Is Not Ideal
The built-in baseline is efficient and readable. The recursive version is pedagogical rather than asymptotically better; it makes endpoint state and stack cost visible while preserving linear copying behavior.

### 9. Optimal Approach
Spread once and validate its length before creating output. Allocate an output array of the same length. Recursively process `(left, right)`: stop when `left > right`; copy the mirrored endpoint pair, then recurse with `left + 1` and `right - 1`. The center of an odd-length string writes to itself.

### 10. Visualization
```text
input:  [A, 😀, B, C]
         L        R   -> output[0] = C, output[3] = A
            L  R      -> output[1] = B, output[2] = 😀
output: [C, B, 😀, A]
```

### 11. Step-by-Step Dry Run
| Call | `left` | `right` | Writes | Output state |
|---:|---:|---:|---|---|
| 1 | 0 | 3 | `0 <- C`, `3 <- A` | `[C, _, _, A]` |
| 2 | 1 | 2 | `1 <- B`, `2 <- 😀` | `[C, B, 😀, A]` |
| 3 | 2 | 1 | base | `[C, B, 😀, A]` |
| return | - | - | join | `"CB😀A"` |

### 12. TypeScript Solution
```typescript
function reverseStringRecursive(text: string): string {
  const codePoints = [...text];
  if (codePoints.length > 1000) {
    throw new RangeError("text must contain at most 1000 Unicode code points");
  }

  const output = new Array<string>(codePoints.length);
  const fill = (left: number, right: number): void => {
    if (left > right) return;
    output[left] = codePoints[right];
    output[right] = codePoints[left];
    fill(left + 1, right - 1);
  };

  fill(0, codePoints.length - 1);
  return output.join("");
}
```

### 13. Code Walkthrough
Spreading precedes the length check because the contract is measured in code points. No output is allocated until validation succeeds. Each helper call fills two mirrored positions and reduces the unresolved interval by two. For empty input, `(0, -1)` hits the base immediately. One final join avoids quadratic recursive concatenation.

### 14. Complexity Analysis
For `n` code points, spreading, filling, and joining each take `O(n)` time. The helper depth is `O(n)` and at most about `n / 2 + 1` frames; the call stack and code-point array use `O(n)` auxiliary space, while the returned string and output array under construction use `O(n)` output-building space.

### 15. Edge Cases
- Empty input returns `""`; one code point returns itself.
- Supplementary code points such as `😀` remain intact.
- Exactly 1,000 code points are accepted; 1,001 throw before output allocation.
- Combining marks and zero-width-joiner sequences are reversed by code point, not preserved as grapheme clusters.
- No normalization occurs, so canonically equivalent strings remain distinct sequences.

### 16. Common Mistakes
- Indexing the original string and splitting surrogate pairs.
- Calling recursive substring and concatenation operations that copy quadratically.
- Measuring UTF-16 `text.length` instead of code-point count.
- Allocating or mutating output before enforcing the depth bound.
- Claiming code-point reversal preserves user-perceived characters.

### 17. Interview Follow-ups
1. Reverse UTF-16 code units deliberately. 2. Reverse grapheme clusters with `Intl.Segmenter`. 3. Reverse words while preserving word spelling. 4. Use an explicit stack.

### 18. Variations
- In-place array reversal. - Reverse words. - Reverse only letters. - Grapheme-aware reversal. - Recursive linked-list reversal.

### 19. Pattern Connections
The inward-moving interval is the same state shape used by palindrome checks and divide-and-conquer partitioning. Separating code-point parsing from recursion establishes a representation invariant the helper can trust.

### 20. NestJS Project Connection
Expose `POST /api/dsa/recursion/reverse-string` with `{ text: string }`. Validate the DTO type, then call only `reverseStringRecursive`; map an oversized code-point sequence to HTTP `400`. Return `{ reversed, trace, truncated }` with at most 200 optimized events such as `{ left, right, leftValue, rightValue, depth }`, and never include a full output snapshot per event. The 1,000-code-point contract limits depth; production grapheme-aware APIs need a different stated contract.

### 21. Mini Challenge
Use `Intl.Segmenter` with grapheme granularity to reverse user-perceived grapheme clusters, then explain how its locale/runtime availability changes the API contract and tests.

## Problem 136 — Palindrome using Recursion

### 1. Problem Statement
Given a string containing at most 1,000 Unicode code points, return whether its code-point sequence is a palindrome. Perform no normalization or case folding. Build `[...text]`, validate the code-point limit before comparison, and throw `RangeError` above `1000`. Empty and single-code-point strings return `true`.

### 2. Example
```text
Input:  "A😀A"
Output: true

Input:  "Aa"
Output: false

Input:  ""
Output: true
```

### 3. Understand the Problem
A palindrome reads identically from both ends under the chosen representation. Here that representation is Unicode code points, with exact case and code-point identity. Compare the outer pair and recurse inward only when it matches.

#### In Simple Terms
If both ends match, move inward; the first mismatch is false, and reaching the middle is true.

### 4. Real-World Analogy
Two readers start at opposite ends of a row of symbols. They compare symbols, step inward together, and stop immediately if their symbols differ.

### 5. Key Observation
After equal endpoints are established, the whole sequence is a palindrome exactly when the smaller interior interval is a palindrome. A mismatch can return early without examining the interior.

### 6. Pattern
```text
Pattern: Recursive Symmetric Comparison
Why this pattern applies: validity depends on equal mirrored endpoints and a smaller center
How to recognize it: compare positions left and right, then shrink both boundaries
Similar problems: palindrome arrays, symmetric trees, mirrored matrix checks
```

### 7. Brute Force
Spread the input into code points, enforce the maximum immediately, create a reversed copy, and compare corresponding entries. Copy before reverse so the original code-point order remains available.

#### Pseudocode
```text
codePoints = spread text into Unicode code points
if length exceeds 1000: throw RangeError
reversed = copied codePoints in reverse order
for each index: if values differ return false
return true
```

#### Brute-Force TypeScript
```typescript
function isPalindromeByReversal(text: string): boolean {
  const codePoints = [...text];
  if (codePoints.length > 1000) {
    throw new RangeError("text must contain at most 1000 Unicode code points");
  }

  const reversed = [...codePoints].reverse();
  return codePoints.every((value, index) => value === reversed[index]);
}
```

#### Visualization
```text
original: [A, 😀, A]
reversed: [A, 😀, A]
compare:   =   =   =   -> true
```

#### Dry Run
For `"A😀A"`, spreading yields three code points. The reversed copy has the same three entries, so every comparison succeeds. For `"Aa"`, index zero compares `"A"` with `"a"` and returns `false`.

#### Brute-Force Complexity
For `n` code points, spreading, reversing, and comparison take `O(n)` time. The code-point array and reversed copy use `O(n)` auxiliary space; the boolean output uses `O(1)` space.

### 8. Why Brute Force Is Not Ideal
The reversed copy duplicates all code-point references even when an early outer mismatch could decide the result. Recursive comparison avoids that second copy, though it still uses linear call-stack space and is mainly pedagogical.

### 9. Optimal Approach
Spread and validate first. Call `compare(0, length - 1)`. Return `true` when `left >= right`, return `false` immediately when endpoint code points differ, and otherwise recurse on `(left + 1, right - 1)`.

### 10. Visualization
```text
[r, a, c, e, c, a, r]
 L                 R     r = r
    L           R        a = a
       L     R           c = c
          L              left >= right -> true
```

### 11. Step-by-Step Dry Run
| Call | `left` | `right` | Compared | Result/state |
|---:|---:|---:|---|---|
| 1 | 0 | 6 | `r` vs `r` | recurse |
| 2 | 1 | 5 | `a` vs `a` | recurse |
| 3 | 2 | 4 | `c` vs `c` | recurse |
| 4 | 3 | 3 | center | `true` |
| unwind | - | - | all prior pairs matched | `true` |

### 12. TypeScript Solution
```typescript
function isPalindromeRecursive(text: string): boolean {
  const codePoints = [...text];
  if (codePoints.length > 1000) {
    throw new RangeError("text must contain at most 1000 Unicode code points");
  }

  const compare = (left: number, right: number): boolean => {
    if (left >= right) return true;
    if (codePoints[left] !== codePoints[right]) return false;
    return compare(left + 1, right - 1);
  };

  return compare(0, codePoints.length - 1);
}
```

### 13. Code Walkthrough
The spread operation defines the code-point representation, and the bound is checked before comparison. The unresolved interval length is the shrinking measure and drops by two after each matching pair. Empty input starts at `(0, -1)` and single input at `(0, 0)`, so both satisfy the true base case. Mismatch returns before another frame is created.

### 14. Complexity Analysis
The code-point conversion takes `O(n)` even when comparison later exits early. Comparison examines at most half the pairs, so total worst-case time is `O(n)`. The code-point array and recursive stack use `O(n)` auxiliary space in the worst case; the boolean output uses `O(1)` space.

### 15. Edge Cases
- Empty and one-code-point inputs return `true`.
- Exact case matters: `"Aa"` is false.
- No normalization occurs: canonically equivalent sequences need not compare equal.
- Supplementary code points remain intact.
- Grapheme clusters can contain multiple code points and are not the comparison unit.
- Exactly 1,000 code points are accepted; 1,001 throw before comparison.

### 16. Common Mistakes
- Normalizing or lowercasing despite the exact contract.
- Comparing UTF-16 code units and splitting supplementary code points.
- Forgetting the `left >= right` base case for even and odd lengths.
- Reversing the only array in place and then comparing it with itself.
- Claiming early mismatch makes total conversion sublinear.

### 17. Interview Follow-ups
1. Ignore punctuation and case under a stated locale. 2. Compare grapheme clusters. 3. Return the first mismatched pair. 4. Check a readonly array generically.

### 18. Variations
- Valid palindrome after one deletion. - Longest palindromic substring. - Palindrome partitioning. - Linked-list palindrome. - Near-palindrome distance.

### 19. Pattern Connections
Palindrome recursion is an inward two-pointer algorithm expressed through the call stack. It also shows why preprocessing cost remains part of total complexity even when the recursive phase can short-circuit.

### 20. NestJS Project Connection
Expose `POST /api/dsa/recursion/palindrome` with `{ text: string }`. Validate DTO shape, call only `isPalindromeRecursive`, and map the code-point bound error to HTTP `400`. Return `{ palindrome, trace, truncated }` with at most 200 optimized events such as `{ left, right, leftValue, rightValue, matched, depth }`; do not trace the reversed-copy baseline or whole input repeatedly. Keep the depth bound explicit and treat normalization or case folding as separate endpoint options with separate contracts.

### 21. Mini Challenge
Return `{ palindrome, mismatch: null | { left, right, leftValue, rightValue } }` recursively while preserving exact code-point indexes and the up-front 1,000-code-point validation rule.

## Problem 137 — Generate Subsequences

### 1. Problem Statement
Given a string containing at most 20 Unicode code points, return all `2^n` positional subsequences as strings. Perform no Unicode normalization. Position choices are distinct, so equal code points can produce duplicate strings. Use deterministic exclude-first depth-first order: for `"ab"`, return `["", "b", "a", "ab"]`. Validate the code-point limit before allocating output.

### 2. Example
```text
Input:  "ab"
Output: ["", "b", "a", "ab"]

Input:  "aa"
Output: ["", "a", "a", "aa"]
```

### 3. Understand the Problem
A subsequence chooses zero or more input positions while preserving their relative order. At each code-point position, the search first excludes that position and then includes it. Different position choices remain different search leaves even when their emitted strings are equal.

#### In Simple Terms
For every code point, first explore leaving it out, then explore putting it in, and record a string after all positions are decided.

### 4. Real-World Analogy
Imagine moving along a row of letter tiles and making two ordered decisions at each tile: walk past it first, then replay that branch while placing it in a tray. Identical-looking tiles still occupy different positions.

### 5. Key Observation
Each of `n` positions has exactly two choices, so there are `2^n` leaves. Emitting a leaf may join as many as `n` code points, making `O(n * 2^n)` time and output size unavoidable in the worst case.

### 6. Pattern
```text
Pattern: Binary Exclude/Include Backtracking
Why this pattern applies: every position creates exactly two ordered choices
How to recognize it: outputs preserve position order but may omit any positions
Similar problems: binary decisions, include/exclude knapsack, subset generation
```

### 7. Brute Force
Spread the string into Unicode code points and enforce the limit. Count from `0` through `2^n - 1`; map the first input position to the highest mask place so increasing masks match exclude-first DFS order. Scan every position for every mask and join the selected code points.

#### Pseudocode
```text
codePoints = spread text into Unicode code points
if length exceeds 20: throw RangeError
for mask from 0 through 2^n - 1:
    selected = empty array
    for each input index:
        if its reversed mask place is set: append that code point
    append join(selected) to results
return results
```

#### Brute-Force TypeScript
```typescript
function generateSubsequencesBitmask(text: string): string[] {
  const codePoints = [...text];
  if (codePoints.length > 20) {
    throw new RangeError("text must contain at most 20 Unicode code points");
  }

  const total = 2 ** codePoints.length;
  const results: string[] = [];
  for (let mask = 0; mask < total; mask += 1) {
    const selected: string[] = [];
    for (let index = 0; index < codePoints.length; index += 1) {
      const place = 2 ** (codePoints.length - 1 - index);
      if (Math.floor(mask / place) % 2 === 1) {
        selected.push(codePoints[index]);
      }
    }
    results.push(selected.join(""));
  }
  return results;
}
```

#### Visualization
```text
mask 00 -> exclude a, exclude b -> ""
mask 01 -> exclude a, include b -> "b"
mask 10 -> include a, exclude b -> "a"
mask 11 -> include a, include b -> "ab"
```

#### Dry Run
For `"ab"`, spreading yields two code points. Masks `0`, `1`, `2`, and `3` select no positions, only index `1`, only index `0`, and both indexes. Joining each selection produces `["", "b", "a", "ab"]`.

#### Brute-Force Complexity
There are `2^n` masks, and each scans `n` positions and can materialize an `O(n)` string, so time is `O(n * 2^n)`. Temporary selection storage is `O(n)` excluding the required `O(n * 2^n)` worst-case output. Both bounds are output-sensitive.

### 8. Why Brute Force Is Not Ideal
The bitmask version already has the optimal output-sensitive asymptotic bound because all results must be materialized. Recursive backtracking is not faster; it exposes the exclude/include decision tree directly, uses one reusable path, and avoids rescanning every earlier position to decide a leaf.

### 9. Optimal Approach
Spread and validate before creating results. Recursively process one code-point index. Explore exclusion first without changing the path, then push the current code point, explore inclusion, and pop it. At `index === n`, join the path and append that fresh string result.

### 10. Visualization
```text
dfs(0, [])
+-- exclude a -> dfs(1, [])
|   +-- exclude b -> ""
|   `-- include b -> "b"
`-- include a -> dfs(1, [a])
    +-- exclude b -> "a"
    `-- include b -> "ab"
```

### 11. Step-by-Step Dry Run
| Event | Index | Choice | Path | Emitted |
|---|---:|---|---|---|
| descend | 0 | exclude `a` | `[]` | - |
| descend | 1 | exclude `b` | `[]` | `""` |
| sibling | 1 | include `b` | `[b]` | `"b"` |
| sibling | 0 | include `a` | `[a]` | - |
| descend | 1 | exclude `b` | `[a]` | `"a"` |
| sibling | 1 | include `b` | `[a, b]` | `"ab"` |

### 12. TypeScript Solution
```typescript
function generateSubsequences(text: string): string[] {
  const codePoints = [...text];
  if (codePoints.length > 20) {
    throw new RangeError("text must contain at most 20 Unicode code points");
  }

  const results: string[] = [];
  const path: string[] = [];
  const search = (index: number): void => {
    if (index === codePoints.length) {
      results.push(path.join(""));
      return;
    }

    search(index + 1);
    path.push(codePoints[index]);
    search(index + 1);
    path.pop();
  };

  search(0);
  return results;
}
```

### 13. Code Walkthrough
Spreading defines `n` in Unicode code points rather than UTF-16 code units, and the maximum is checked before either output array is allocated. The undecided-position count `n - index` decreases on every recursive edge. Exclusion runs before inclusion, fixing the public order. Only the local path mutates; every push is undone, and each leaf joins the current path into an independent immutable string.

### 14. Complexity Analysis
The decision tree has `2^n` leaves. Joining paths across those leaves costs `O(n * 2^n)` in the worst case, matching the size of the returned strings; search overhead is smaller than that output-materialization bound. The path and recursion stack use `O(n)` auxiliary space excluding `O(n * 2^n)` output space. The 20-code-point bound limits depth and the exponential result count.

### 15. Edge Cases
- Empty input returns `[""]`.
- `"aa"` returns two separate `"a"` entries because positions are distinct.
- Supplementary code points such as `😀` remain intact.
- Combining sequences are choices of code points, not grapheme clusters.
- Exactly 20 code points are accepted; 21 throw before output allocation.
- No normalization occurs, so canonically equivalent inputs remain distinct sequences.

### 16. Common Mistakes
- Iterating UTF-16 code units with string indexes.
- Deduplicating equal output strings despite positional semantics.
- Running inclusion before exclusion and changing the required order.
- Forgetting to pop the included code point before returning to a sibling.
- Claiming `O(2^n)` time while ignoring string materialization.
- Starting output generation before enforcing the size bound.

### 17. Interview Follow-ups
1. Emit only subsequences of length `k`. 2. Stream results lazily. 3. Deduplicate equal strings under a changed contract. 4. Use grapheme clusters as positions.

### 18. Variations
- Increasing subsequences. - Subsequences summing to a target. - Longest common subsequence. - Distinct subsequence counting. - Lazy generators.

### 19. Pattern Connections
This is the direct binary decision-tree form of backtracking. It also demonstrates output-sensitive analysis: recursion can reduce repeated decision bookkeeping, but no eager algorithm can avoid constructing the exponentially large required return value.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/subsequences` with `{ text: string }`. Validate the DTO and code-point maximum, then call only `generateSubsequences`; map `RangeError` to HTTP `400`. Return the result plus at most 200 optimized trace events such as `{ index, depth, action: "exclude" | "include" | "emit", pathLength }` and a `truncated` flag. Do not attach the path or full output snapshot to every event. The input bound limits depth, but response-size policy may need a stricter production cap.

### 21. Mini Challenge
Return only subsequences containing exactly `k` code points, pruning when the current path is already too long or too few undecided positions remain, while preserving exclude-first order.

## Problem 138 — Generate Subsets

### 1. Problem Statement
Given a readonly array of at most 20 generic values, return every positional subset as a fresh array without mutating the input. Preserve input order inside each subset and use start-index depth-first output order: emit the current path, then choose each remaining position from left to right. Thus `[a, b]` returns `[[], [a], [a, b], [b]]`. Equal values at different positions are distinct choices, so duplicate-valued subsets are allowed.

### 2. Example
```text
Input:  ["a", "b"]
Output: [[], ["a"], ["a", "b"], ["b"]]

Input:  [1, 1]
Output: [[], [1], [1, 1], [1]]
```

### 3. Understand the Problem
Here a subset means a choice of array positions. Each emitted array preserves the chosen positions' input order, but the collection itself follows a separate explicit DFS preorder. This positional contract differs from mathematical-set deduplication and from binary include/exclude collection order.

#### In Simple Terms
Save the choices made so far, then try each later position as the next choice and backtrack before trying its sibling.

### 4. Real-World Analogy
At a buffet arranged left to right, photograph the current tray first, then add each dish that appears after the last chosen dish. Remove it before trying the next dish so every photograph owns its own selection.

### 5. Key Observation
Every subset has one unique increasing sequence of chosen indexes. Emitting on entry and recursing only on later indexes visits each of the `2^n` positional subsets exactly once in the required order.

### 6. Pattern
```text
Pattern: Start-Index Backtracking with Preorder Emission
Why this pattern applies: each next choice must come from a later position
How to recognize it: paths preserve input order and may stop at any depth
Similar problems: combinations, combination sum, increasing index selections
```

### 7. Brute Force
Use a bitmask for each positional subset. To preserve the public start-index DFS order, convert each output rank into its corresponding mask by skipping whole `2^remaining` DFS blocks. Then scan that mask from left to right and push a fresh selected array.

#### Pseudocode
```text
validate input length is at most 20
for rank from 0 through 2^n - 1:
    derive the positional mask at that start-index DFS rank
    scan input indexes and copy values whose bits are set
    append the fresh selected array
return results
```

#### Brute-Force TypeScript
```typescript
function generateSubsetsBitmask<T>(values: readonly T[]): T[][] {
  if (values.length > 20) {
    throw new RangeError("values must contain at most 20 items");
  }

  const maskAtRank = (rank: number): number => {
    let remainingRank = rank;
    let start = 0;
    let mask = 0;
    while (remainingRank > 0) {
      for (let index = start; index < values.length; index += 1) {
        const blockSize = 2 ** (values.length - index - 1);
        if (remainingRank > blockSize) {
          remainingRank -= blockSize;
          continue;
        }
        mask += 2 ** index;
        remainingRank -= 1;
        start = index + 1;
        break;
      }
    }
    return mask;
  };

  const results: T[][] = [];
  const total = 2 ** values.length;
  for (let rank = 0; rank < total; rank += 1) {
    const mask = maskAtRank(rank);
    const selected: T[] = [];
    for (let index = 0; index < values.length; index += 1) {
      if (Math.floor(mask / (2 ** index)) % 2 === 1) {
        selected.push(values[index]);
      }
    }
    results.push(selected);
  }
  return results;
}
```

#### Visualization
```text
rank 0 -> mask 00 -> []
rank 1 -> mask 01 -> [a]
rank 2 -> mask 11 -> [a, b]
rank 3 -> mask 10 -> [b]
```

#### Dry Run
For `[a, b]`, rank zero maps to no positions. The next DFS block starts with index zero, yielding masks for `[a]` and `[a, b]`. The final block starts with index one, yielding `[b]`. Every result is a newly allocated array.

#### Brute-Force Complexity
There are `2^n` ranks. Mask derivation and mask scanning each take `O(n)` per result, so time is `O(n * 2^n)`. Temporary mask and selected-array work is `O(n)` excluding the required `O(n * 2^n)` worst-case output. The complexity includes copying fresh result arrays.

### 8. Why Brute Force Is Not Ideal
The bitmask baseline meets the order and asymptotic output bound, but rank-to-mask conversion obscures the simple structure of the contract. Start-index backtracking produces the same preorder directly, reuses one local path, and naturally explains why elements inside every emitted subset remain in input order. It is clearer, not asymptotically faster.

### 9. Optimal Approach
Validate the length before creating output. Define `search(start)`: first append `[...path]`, then loop `index` from `start` to the end. Push `values[index]`, recurse with `index + 1`, and pop before the next iteration. Entry-time emission produces `[]`, `[a]`, `[a, b]`, `[b]` for `[a, b]`.

### 10. Visualization
```text
emit []
+-- choose a -> emit [a]
|   `-- choose b -> emit [a, b]
`-- choose b -> emit [b]
```

### 11. Step-by-Step Dry Run
| Event | `start` | Choice | Path | Output appended |
|---|---:|---|---|---|
| enter | 0 | - | `[]` | `[]` |
| choose | 0 | index 0 | `[a]` | `[a]` |
| choose | 1 | index 1 | `[a, b]` | `[a, b]` |
| unchoose | 1 | index 1 | `[a]` | - |
| unchoose | 0 | index 0 | `[]` | - |
| choose | 0 | index 1 | `[b]` | `[b]` |

### 12. TypeScript Solution
```typescript
function generateSubsets<T>(values: readonly T[]): T[][] {
  if (values.length > 20) {
    throw new RangeError("values must contain at most 20 items");
  }

  const results: T[][] = [];
  const path: T[] = [];
  const search = (start: number): void => {
    results.push([...path]);
    for (let index = start; index < values.length; index += 1) {
      path.push(values[index]);
      search(index + 1);
      path.pop();
    }
  };

  search(0);
  return results;
}
```

### 13. Code Walkthrough
The readonly parameter and absence of writes to `values` protect the caller's array. Each recursive edge advances `start`, bounding depth by `n`. Spreading `path` at every entry creates a fresh output snapshot before local mutation continues. The loop's left-to-right order and entry-time emission establish the exact collection order; popping restores the path for every sibling.

### 14. Complexity Analysis
The function emits `2^n` arrays. Across all snapshots it copies `Theta(n * 2^n)` values in the worst case, so time and required output space are `O(n * 2^n)` and output-sensitive. The recursion stack and reusable path use `O(n)` auxiliary space excluding output. The bound of 20 limits both depth and eager response size.

### 15. Edge Cases
- Empty input returns `[[]]`.
- Every result array is a fresh top-level snapshot; changing one result does not change another.
- Input values themselves are not deep-cloned, so object elements retain their original identities.
- `[1, 1]` emits two distinct one-position subsets that both display as `[1]`.
- Exactly 20 values are accepted; 21 throw before output allocation.
- The readonly input is never sorted, spliced, or otherwise mutated.

### 16. Common Mistakes
- Pushing `path` itself instead of `[...path]`, causing result aliasing.
- Deduplicating equal values and losing positional subsets.
- Calling these outputs subsequences without stating the positional subset contract and collection order.
- Using binary include/exclude order instead of the required start-index preorder.
- Recursing with `start + 1` instead of `index + 1`.
- Ignoring snapshot copying in the complexity analysis.

### 17. Interview Follow-ups
1. Return only subsets of size `k`. 2. Deduplicate equal-valued subsets under a changed contract. 3. Stream snapshots lazily. 4. Return selected indexes instead of values.

### 18. Variations
- Combinations. - Power sets with duplicate suppression. - Combination sum. - Increasing subsequences. - Bitset representations.

### 19. Pattern Connections
Start-index recursion is the standard bridge from subset generation to combinations. Unlike binary subsequence recursion, a call represents a completed subset plus all ways to extend it, so preorder emission defines a different but equally deterministic output sequence.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/subsets` with `{ values: unknown[] }`. Validate array shape and the 20-item maximum, then call only `generateSubsets`; map `RangeError` to HTTP `400`. Return subsets plus at most 200 optimized events such as `{ start, chosenIndex, depth, action: "emit" | "choose" | "unchoose", pathLength }` and `truncated`. Never include the path or accumulating output in every event. Apply JSON payload and response limits because shallow algorithmic snapshots can still contain large caller objects.

### 21. Mini Challenge
Generate only size-`k` subsets in the same left-to-right order, pruning when `path.length + (values.length - start) < k`, and prove that every emitted array is still a fresh snapshot.

## Problem 139 — Generate Permutations

### 1. Problem Statement
Given a readonly array of at most 9 generic values, return all `n!` permutations in deterministic input-order choice DFS order. Values must be pairwise distinct under JavaScript `Set` SameValueZero semantics; throw `RangeError` for any duplicate before allocating output. `NaN` equals `NaN`, and `0` equals `-0` for this validation, while objects are equal only by identity. Return fresh arrays, do not mutate the input, and return `[[]]` for empty input.

### 2. Example
```text
Input:  ["a", "b", "c"]
Output: [
  ["a", "b", "c"], ["a", "c", "b"],
  ["b", "a", "c"], ["b", "c", "a"],
  ["c", "a", "b"], ["c", "b", "a"]
]

Input:  []
Output: [[]]
```

### 3. Understand the Problem
A permutation uses every input position exactly once. Because values are validated as distinct under SameValueZero, choosing unused indexes also guarantees no duplicate-valued outputs. At each depth, try candidate indexes from zero upward to establish deterministic input-order DFS output.

#### In Simple Terms
Choose each unused input item as the next item, recurse, then undo that choice so the next candidate can use the same position later.

### 4. Real-World Analogy
Arrange distinct name cards into every possible seating row. Fill the first empty seat by trying cards in their original order, finish the remaining seats, then remove the card and try the next one.

### 5. Key Observation
At depth `d`, exactly `n - d` indexes remain available. A boolean used-index array makes membership checks constant time, while copying each complete length-`n` path accounts for the unavoidable `Theta(n * n!)` output size.

### 6. Pattern
```text
Pattern: Used-Index Permutation Backtracking
Why this pattern applies: each position must be chosen exactly once
How to recognize it: output length is fixed and candidate order defines permutation order
Similar problems: anagrams, assignment search, Hamiltonian path state
```

### 7. Brute Force
Validate length and SameValueZero distinctness first. Enumerate all `n^n` base-`n` index sequences of length `n`, decoding the most significant digit first. Reject a sequence as soon as an index repeats; map each repetition-free sequence to a fresh permutation. Handle `n = 0` separately because its one valid permutation is the empty array.

#### Pseudocode
```text
validate length at most 9 and values pairwise distinct with Set
if input is empty: return [[]]
for encoded from 0 through n^n - 1:
    decode n base-n indexes from most significant to least
    reject when an index repeats
    otherwise map indexes to a fresh permutation and append it
return results
```

#### Brute-Force TypeScript
```typescript
function generatePermutationsBrute<T>(values: readonly T[]): T[][] {
  if (values.length > 9) {
    throw new RangeError("values must contain at most 9 items");
  }
  if (new Set<T>(values).size !== values.length) {
    throw new RangeError("values must be pairwise distinct under SameValueZero");
  }
  if (values.length === 0) return [[]];

  const results: T[][] = [];
  const sequenceCount = values.length ** values.length;
  for (let encoded = 0; encoded < sequenceCount; encoded += 1) {
    const used = new Array<boolean>(values.length).fill(false);
    const permutation: T[] = [];
    let valid = true;
    for (let depth = 0; depth < values.length; depth += 1) {
      const place = values.length ** (values.length - 1 - depth);
      const index = Math.floor(encoded / place) % values.length;
      if (used[index]) {
        valid = false;
        break;
      }
      used[index] = true;
      permutation.push(values[index]);
    }
    if (valid) results.push(permutation);
  }
  return results;
}
```

#### Visualization
```text
base-3 indexes for [a, b, c]
000 -> reject repeated 0
001 -> reject repeated 0
...
012 -> [a, b, c]
021 -> [a, c, b]
...
210 -> [c, b, a]
```

#### Dry Run
For `[a, b]`, the four base-two sequences are `00`, `01`, `10`, and `11`. The repeated-index sequences `00` and `11` are rejected. `01` emits `[a, b]`, and `10` emits `[b, a]`, matching input-order DFS.

#### Brute-Force Complexity
The baseline considers `n^n` index sequences and examines at most `n` digits for each, taking `O(n * n^n)` time. Its temporary used array and candidate permutation use `O(n)` auxiliary space. Valid outputs require `Theta(n * n!)` space. Early rejection helps actual work but does not improve the stated worst-case bound.

### 8. Why Brute Force Is Not Ideal
Most length-`n` index sequences repeat an index and can never be permutations. Backtracking prevents those invalid prefixes from being created at all. The optimized search still must materialize `n!` fresh arrays, but it visits only prefixes that can extend to valid permutations.

### 9. Optimal Approach
Validate the maximum and `Set` size before allocating output. Maintain a reusable path and `used[index]` flags. At each depth, loop indexes in input order, skip used indexes, mark and push a candidate, recurse, then pop and unmark it. When path length reaches `n`, append `[...path]`.

### 10. Visualization
```text
[]
+-- a -> [a]
|   +-- b -> [a, b] -> [a, b, c]
|   `-- c -> [a, c] -> [a, c, b]
+-- b -> [b] -> [b, a, c], [b, c, a]
`-- c -> [c] -> [c, a, b], [c, b, a]
```

### 11. Step-by-Step Dry Run
| Event | Depth | Chosen index | Path | Result |
|---|---:|---:|---|---|
| choose | 0 | 0 | `[a]` | - |
| choose | 1 | 1 | `[a, b]` | - |
| choose | 2 | 2 | `[a, b, c]` | emit copy |
| backtrack | 1 | 1 | `[a]` | - |
| choose | 1 | 2 | `[a, c]` | continue with `b` |
| backtrack | 0 | 0 | `[]` | next root choice is `b` |

### 12. TypeScript Solution
```typescript
function generatePermutations<T>(values: readonly T[]): T[][] {
  if (values.length > 9) {
    throw new RangeError("values must contain at most 9 items");
  }
  if (new Set<T>(values).size !== values.length) {
    throw new RangeError("values must be pairwise distinct under SameValueZero");
  }

  const results: T[][] = [];
  const path: T[] = [];
  const used = new Array<boolean>(values.length).fill(false);
  const search = (): void => {
    if (path.length === values.length) {
      results.push([...path]);
      return;
    }

    for (let index = 0; index < values.length; index += 1) {
      if (used[index]) continue;
      used[index] = true;
      path.push(values[index]);
      search();
      path.pop();
      used[index] = false;
    }
  };

  search();
  return results;
}
```

### 13. Code Walkthrough
The two validations finish before results, path, or used arrays are allocated. `Set` applies SameValueZero: repeated `NaN` and mixed `0`/`-0` are duplicates, while separate object instances are distinct. Candidate iteration fixes output order. The helper mutates only local state, restores both path and used flag after each child, and emits a fresh top-level array at every leaf. Empty input reaches the leaf immediately and returns `[[]]`.

### 14. Complexity Analysis
There are `n!` outputs, and copying `n` values into each fresh result requires `Theta(n * n!)` time and output space. Prefix traversal is within that bound for nonempty `n`. The path, used array, and recursion stack use `O(n)` auxiliary space excluding output. The maximum of 9 bounds recursion depth and the eager factorial response.

### 15. Edge Cases
- Empty input returns `[[]]`; one value returns `[[value]]`.
- Two `NaN` values are duplicates and throw before output allocation.
- `0` and `-0` are SameValueZero duplicates and throw.
- Repeating the same object reference throws; two distinct objects are allowed.
- Result arrays are fresh snapshots, but contained object values retain identity.
- Exactly 9 values are accepted; 10 throw before duplicate checking output work.

### 16. Common Mistakes
- Swapping elements in the caller's array and violating readonly ownership.
- Checking duplicates with `===`, which does not treat `NaN` as equal to itself.
- Treating two structurally equal but separately allocated objects as duplicates.
- Pushing the mutable path without copying it.
- Forgetting to clear a used flag after recursion.
- Calling permutation generation `O(n!)` while ignoring length-`n` output copies.

### 17. Interview Follow-ups
1. Allow duplicates and emit unique permutations. 2. Return the `k`th permutation without listing all prior results. 3. Stream permutations lazily. 4. Stop after a caller-supplied result limit.

### 18. Variations
- String anagrams. - Unique multiset permutations. - Next permutation. - Assignment problems. - Permutations of length `k`.

### 19. Pattern Connections
Used-index backtracking is depth-first search over valid partial assignments. It replaces brute-force generate-and-filter work with constraint maintenance, while snapshot ownership and output-sensitive cost remain the same concerns seen in subset generation.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/permutations` with `{ values: unknown[] }`. Validate shape, length, and SameValueZero distinctness, then call only `generatePermutations`; map `RangeError` to HTTP `400`. Return permutations plus at most 200 optimized events such as `{ depth, chosenIndex, action: "choose" | "unchoose" | "emit", pathLength }` and `truncated`. Never serialize the full path, used array, or accumulating output per event. A production endpoint should usually impose a result or response-byte limit below the algorithmic maximum.

### 21. Mini Challenge
Change the contract to permit duplicate values and emit only unique permutations in deterministic order, explaining how SameValueZero grouping, per-depth duplicate skipping, and object identity affect the design.
## Problem 140 — Combination Sum

### 1. Problem Statement
Given a readonly array of at most 20 pairwise-distinct positive safe integers and a safe-integer target from 0 through 1000, return every unique combination whose sum is the target. A candidate may be reused any number of times. Values within each combination must be in nondecreasing numeric order, and combinations must be returned in lexicographic order according to the ascending candidate choices. Validate the entire input before copying and sorting the candidates; do not mutate the caller's array. Return `[[]]` when the target is zero, return `[]` when the target is positive and there are no candidates, and throw `RangeError` for duplicate or otherwise invalid candidates.

### 2. Example
```text
Input:  candidates = [2, 3, 6, 7], target = 7
Output: [[2, 2, 3], [7]]

Input:  candidates = [8, 3, 5], target = 0
Output: [[]]
```

### 3. Understand the Problem
Each candidate is a reusable choice, but different orders of the same values do not create different combinations. Sorting a private copy gives every recursive level a stable numeric choice order. Keeping a start index then prevents permutations such as `[2, 3, 2]` while still allowing `[2, 2, 3]` by recursing with the same index.

#### In Simple Terms
Choose numbers from left to right in a sorted private copy, allow the current number to be chosen again, and save a copy whenever the remaining target becomes zero.

### 4. Real-World Analogy
Imagine making an exact-value bundle from unlimited supplies of distinct coin denominations. Once a bundle starts using a larger denomination, never return to a smaller one; this gives every bundle one canonical construction order.

### 5. Key Observation
Positive candidates make the remaining sum strictly decrease after every choice. If a sorted candidate exceeds the remaining amount, it and every later candidate are impossible, so the loop can stop. Reusing an index rather than advancing past it models unlimited reuse.

### 6. Pattern
```text
Pattern: Start-Index Backtracking with Reusable Choices
Why this pattern applies: combinations ignore order, but each candidate can be reused
How to recognize it: choose from a suffix and recurse from the chosen index
Similar problems: coin change enumeration, factor combinations, bounded sum search
```

### 7. Brute Force
Validate first, then copy and sort ascending. For positive targets and nonempty candidates, let `D = floor(target / minimumCandidate)`. Enumerate every ordered candidate-index sequence of every length from zero through `D`, without pruning when a partial sum exceeds the target. Whenever a sequence sums to the target, count its indexes, expand those counts into a sorted value sequence, serialize that canonical sequence as a key, and use a `Set` to suppress duplicate permutations. Ascending depth-first index enumeration discovers each canonical sorted sequence before any of its nonsorted permutations, so first insertion order is the required lexicographic combination order.

#### Pseudocode
```text
validate target, length, positive safe integers, and distinct values
copy and sort candidates ascending
if target is zero: return [[]]
if candidates are empty: return []
D = floor(target / smallest candidate)
visit every ordered index sequence with length 0 through D:
    if its sum equals target:
        canonicalize its counts as a sorted value sequence
        add it only when its serialized key has not been seen
    if its length is less than D:
        append every candidate index in ascending order and recurse
return results in Set discovery order
```

#### Brute-Force TypeScript
```typescript
function combinationSumBrute(
  candidates: readonly number[],
  target: number,
): number[][] {
  if (!Number.isSafeInteger(target) || target < 0 || target > 1000) {
    throw new RangeError("target must be a safe integer from 0 through 1000");
  }
  if (candidates.length > 20) {
    throw new RangeError("candidates must contain at most 20 values");
  }

  const distinct = new Set<number>();
  for (const candidate of candidates) {
    if (!Number.isSafeInteger(candidate) || candidate <= 0) {
      throw new RangeError("candidates must be positive safe integers");
    }
    if (distinct.has(candidate)) {
      throw new RangeError("candidates must have distinct numeric values");
    }
    distinct.add(candidate);
  }

  const sorted = [...candidates].sort((left, right) => left - right);
  if (target === 0) return [[]];
  if (sorted.length === 0) return [];

  const maximumDepth = Math.floor(target / sorted[0]);
  const results: number[][] = [];
  const seen = new Set<string>();
  const indexPath: number[] = [];
  const enumerate = (sum: number): void => {
    if (sum === target) {
      const counts = new Array<number>(sorted.length).fill(0);
      for (const index of indexPath) counts[index] += 1;

      const canonical: number[] = [];
      for (let index = 0; index < sorted.length; index += 1) {
        for (let count = 0; count < counts[index]; count += 1) {
          canonical.push(sorted[index]);
        }
      }
      const key = JSON.stringify(canonical);
      if (!seen.has(key)) {
        seen.add(key);
        results.push(canonical);
      }
    }
    if (indexPath.length === maximumDepth) return;

    for (let index = 0; index < sorted.length; index += 1) {
      indexPath.push(index);
      enumerate(sum + sorted[index]);
      indexPath.pop();
    }
  };

  enumerate(0);
  return results;
}
```

#### Visualization
```text
all ordered sequences for [2, 3], target 6, D = 3
[]
+-- [2] -> [2, 2] -> [2, 2, 2] => canonical [2, 2, 2]
|                       `-- [2, 2, 3]
+-- [3] -> [3, 2] -> ...
|          `-- [3, 3] => canonical [3, 3]
`-- continue every branch through depth 3, even after sum > 6
```

#### Dry Run
For `[2, 3, 6, 7]` and target `7`, the maximum depth is three. The ordered sequence `[2, 2, 3]` is the first target-sum sequence and inserts canonical `[2, 2, 3]`; later permutations reuse its key. The one-item sequence `[7]` inserts `[7]`. Every sequence still extends until depth three, even if its sum has already exceeded seven.

#### Brute-Force Complexity
Let `b` be the candidate count and `D = floor(target / minimumCandidate)`. The baseline visits `1 + b + ... + b^D` sequences and may spend `O(D)` canonicalizing a target-sum sequence, for `O(b^D * D)` time as a convenient worst-case bound. Its recursion and index path use `O(D)` auxiliary space; the deduplication keys and returned snapshots use `O(S * D)` space for `S` unique combinations. The baseline is intentionally impractical when `D` is large.

### 8. Why Brute Force Is Not Ideal
Ordered enumeration generates permutations that represent the same combination, continues branches that positive values have already made too large, and explores every depth through `D`. Start-index backtracking creates only nondecreasing paths, while remaining-sum pruning rejects impossible extensions immediately.

### 9. Optimal Approach
Validate every input before output allocation, copy and sort the candidates, then run a depth-first search with `(startIndex, remaining)`. Emit a fresh path snapshot when `remaining` is zero. Iterate from `startIndex` upward; because the array is sorted, break when a candidate exceeds `remaining`. Push the candidate, recurse with the same index to permit reuse, then pop it. Subtract only after `candidate <= remaining`, so the new remaining value stays within the safe integer range `0..1000`.

### 10. Visualization
```text
[2, 3, 6, 7], remaining 7
[]
+-- 2, remaining 5
|   +-- 2, remaining 3
|   |   `-- 3, remaining 0 -> [2, 2, 3]
|   `-- 3, remaining 2 -> next candidate is too large
+-- 3, remaining 4 -> no completion
+-- 6, remaining 1 -> stop
`-- 7, remaining 0 -> [7]
```

### 11. Step-by-Step Dry Run
| Event | Start index | Remaining | Path | Result |
|---|---:|---:|---|---|
| choose `2` | 0 | 5 | `[2]` | - |
| choose `2` | 0 | 3 | `[2, 2]` | - |
| choose `3` | 1 | 0 | `[2, 2, 3]` | emit copy |
| unchoose to root | 0 | 7 | `[]` | `[[2, 2, 3]]` |
| choose `7` | 3 | 0 | `[7]` | emit copy |

### 12. TypeScript Solution
```typescript
function combinationSum(
  candidates: readonly number[],
  target: number,
): number[][] {
  if (!Number.isSafeInteger(target) || target < 0 || target > 1000) {
    throw new RangeError("target must be a safe integer from 0 through 1000");
  }
  if (candidates.length > 20) {
    throw new RangeError("candidates must contain at most 20 values");
  }

  const distinct = new Set<number>();
  for (const candidate of candidates) {
    if (!Number.isSafeInteger(candidate) || candidate <= 0) {
      throw new RangeError("candidates must be positive safe integers");
    }
    if (distinct.has(candidate)) {
      throw new RangeError("candidates must have distinct numeric values");
    }
    distinct.add(candidate);
  }

  const sorted = [...candidates].sort((left, right) => left - right);
  const results: number[][] = [];
  const path: number[] = [];
  const search = (startIndex: number, remaining: number): void => {
    if (remaining === 0) {
      results.push([...path]);
      return;
    }

    for (let index = startIndex; index < sorted.length; index += 1) {
      const candidate = sorted[index];
      if (candidate > remaining) break;
      path.push(candidate);
      search(index, remaining - candidate);
      path.pop();
    }
  };

  search(0, target);
  return results;
}
```

### 13. Code Walkthrough
Target, length, value, and duplicate validation all complete before the private sorted copy and output arrays are created. Sorting fixes numeric output order without touching the readonly input. A recursive call receives the chosen index again, allowing unlimited reuse; later calls can never move left, so every path is nondecreasing and unique. The sorted loop's break is valid because all later values are at least as large. Each result uses `[...path]`, so later backtracking cannot change an emitted combination. A zero target emits the initial empty path, while a positive target with no candidates finishes with `[]`.

### 14. Complexity Analysis
Let `N` count the candidate examinations across all reachable nondecreasing search nodes, let `D = floor(target / minimumCandidate)` for nonempty positive-target input, and let `S` be the result count. Traversal plus snapshot materialization takes `O(N + S * D)` time, conservatively `O(N * D)`; `N` is exponential in the worst case because choices may be reused. Returned snapshots occupy `O(S * D)` output space. The path and recursion stack use `O(D)` auxiliary space, while the sorted copy and validation set use `O(b)` space. There is no meaningful `O(2^b)` bound because reuse makes solution depth depend on the target, not just candidate count.

### 15. Edge Cases
- Target zero returns `[[]]` after full candidate validation.
- A positive target with an empty candidate array returns `[]`.
- Duplicate numeric values throw, even when they are not needed for the target.
- Zero, negative, fractional, infinite, `NaN`, or unsafe candidates throw.
- An invalid target or more than 20 candidates throws before search.
- Candidates larger than the remaining sum stop the sorted loop.
- The input order and values remain unchanged.

### 16. Common Mistakes
- Sorting the caller's array in place.
- Advancing to `index + 1` and accidentally forbidding reuse.
- Recursing to an earlier index and producing permutation duplicates.
- Using `continue` instead of the stronger sorted-order `break`.
- Subtracting before checking whether a large candidate exceeds the remaining amount.
- Claiming `O(2^b)` despite unlimited reuse.
- Pushing the mutable path without a fresh copy.

### 17. Interview Follow-ups
1. Permit each candidate at most once. 2. Accept duplicate candidates while returning unique combinations. 3. Return only the number of combinations. 4. Stream combinations with a result limit.

### 18. Variations
- Combination Sum II. - Coin change counting. - Fixed-length combination sums. - Bounded candidate multiplicities. - Factor combinations.

### 19. Pattern Connections
This is subset-style start-index DFS with one important change: recursion keeps the chosen index instead of advancing beyond it. Positive-number remaining-sum pruning turns sorted order into both a uniqueness rule and an early termination rule.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/combination-sum` with `{ candidates: unknown[], target: unknown }`. Validate DTO shape, numeric limits, positivity, safe integers, and duplicate values, then call only `combinationSum`; map `RangeError` to HTTP `400`. Return combinations plus at most 200 optimized events such as `{ startIndex, chosenIndex, remaining, depth, action: "choose" | "unchoose" | "emit" }` and `truncated`. Never attach the full path or accumulated results to every event. Enforce response-byte and result-count limits because valid output can be exponential even under the input caps.

### 21. Mini Challenge
Add an optional positive safe-integer `maximumLength` and prune before choosing when `path.length === maximumLength`, while preserving lexicographic output and the target-zero behavior.

## Problem 141 — Letter Combinations of Phone Number

### 1. Problem Statement
Given a string of at most 10 ASCII digits containing only `'2'` through `'9'`, return every letter combination represented by a standard telephone keypad. Return `[]` for the empty string. Throw `RangeError` for excessive length or any invalid character before producing output. Preserve digit order, use each digit's letters in standard left-to-right order, and return combinations in deterministic Cartesian-product order.

### 2. Example
```text
Input:  "23"
Output: ["ad", "ae", "af", "bd", "be", "bf", "cd", "ce", "cf"]

Input:  ""
Output: []
```

### 3. Understand the Problem
Every input position contributes one character from its keypad mapping. The first digit changes slowest, and the final digit changes fastest, which is exactly the order produced by nested Cartesian-product loops or a depth-first search that tries mapped letters from left to right.

#### In Simple Terms
Pick one letter for the first digit, then one for the next digit, continuing until one complete word has one letter per digit.

### 4. Real-World Analogy
Treat each digit as a wheel printed with three or four letters. List every display by turning the rightmost wheel through all its letters before advancing the wheel to its left.

### 5. Key Observation
The output size is the product of the mapping lengths, at most `4^n`. Every result has length `n`, so merely materializing the output costs `Theta(outputCount * n)` time and space. ASCII subtraction maps digit `'2'` to array index zero without object lookup ambiguity.

### 6. Pattern
```text
Pattern: Cartesian-Product Backtracking
Why this pattern applies: each input position contributes one independent ordered choice
How to recognize it: fixed depth, one choice group per level, every leaf is valid
Similar problems: keypad words, product configurations, character substitution
```

### 7. Brute Force
Validate the complete string first. Start with the single empty prefix. For each digit from left to right, allocate a new list and concatenate every current prefix with every mapped letter in standard order. Replace the current list after the whole level is built. This explicitly materializes every intermediate Cartesian-product layer.

#### Pseudocode
```text
validate length at most 10 and every character from '2' through '9'
if digits is empty: return []
combinations = [empty string]
for each digit from left to right:
    next = []
    for each existing prefix in order:
        for each mapped letter in order:
            append prefix + letter to next
    combinations = next
return combinations
```

#### Brute-Force TypeScript
```typescript
function letterCombinationsBrute(digits: string): string[] {
  if (digits.length > 10) {
    throw new RangeError("digits must contain at most 10 characters");
  }
  for (const digit of digits) {
    if (digit < "2" || digit > "9") {
      throw new RangeError("digits must contain only ASCII characters '2' through '9'");
    }
  }
  if (digits.length === 0) return [];

  const mapping: readonly string[] = [
    "abc",
    "def",
    "ghi",
    "jkl",
    "mno",
    "pqrs",
    "tuv",
    "wxyz",
  ];
  let combinations: string[] = [""];
  for (const digit of digits) {
    const mappingIndex = digit.charCodeAt(0) - "2".charCodeAt(0);
    const letters = mapping[mappingIndex];
    const next: string[] = [];
    for (const prefix of combinations) {
      for (const letter of letters) next.push(prefix + letter);
    }
    combinations = next;
  }
  return combinations;
}
```

#### Visualization
```text
digits "23"
[""]
  x [a, b, c] -> [a, b, c]
  x [d, e, f] -> [ad, ae, af, bd, be, bf, cd, ce, cf]
```

#### Dry Run
For digit `2`, the empty prefix expands to `a`, `b`, and `c`. For digit `3`, prefix `a` produces `ad`, `ae`, `af`; then `b` produces `bd`, `be`, `bf`; then `c` produces `cd`, `ce`, `cf`. The loops therefore establish the required deterministic order.

#### Brute-Force Complexity
For `n` digits, there are at most `4^n` final strings of length `n`. Building and copying strings across the iterative layers takes `O(4^n * n)` time and the returned output takes `O(4^n * n)` space. Two adjacent product layers may coexist, so intermediate storage is also output-scale.

### 8. Why Brute Force Is Not Ideal
The iterative product is already asymptotically optimal because it must create every output string. A recursive formulation is still preferable when teaching or extending backtracking: each depth directly models one digit's choices, uses one reusable character path, and can later support streaming, limits, or prefix constraints without storing a complete intermediate layer.

### 9. Optimal Approach
Validate every character before allocating results. Return early for the empty input. Use a character path of length at most `n`; at depth `d`, obtain the mapping with `digits.charCodeAt(d) - 50`, try its letters in order, recurse to `d + 1`, and pop. At depth `n`, join the path into a fresh immutable string and append it.

### 10. Visualization
```text
"23"
[]
+-- a
|   +-- d -> ad
|   +-- e -> ae
|   `-- f -> af
+-- b -> bd, be, bf
`-- c -> cd, ce, cf
```

### 11. Step-by-Step Dry Run
| Event | Digit index | Choice | Path | Output |
|---|---:|---|---|---|
| choose | 0 | `a` | `[a]` | - |
| choose | 1 | `d` | `[a, d]` | `ad` |
| replace last choice | 1 | `e` | `[a, e]` | `ae` |
| replace last choice | 1 | `f` | `[a, f]` | `af` |
| backtrack and choose | 0 | `b` | `[b]` | next outputs start with `b` |

### 12. TypeScript Solution
```typescript
function letterCombinations(digits: string): string[] {
  if (digits.length > 10) {
    throw new RangeError("digits must contain at most 10 characters");
  }
  for (const digit of digits) {
    if (digit < "2" || digit > "9") {
      throw new RangeError("digits must contain only ASCII characters '2' through '9'");
    }
  }
  if (digits.length === 0) return [];

  const mapping: readonly string[] = [
    "abc",
    "def",
    "ghi",
    "jkl",
    "mno",
    "pqrs",
    "tuv",
    "wxyz",
  ];
  const results: string[] = [];
  const path: string[] = [];
  const search = (digitIndex: number): void => {
    if (digitIndex === digits.length) {
      results.push(path.join(""));
      return;
    }

    const mappingIndex = digits.charCodeAt(digitIndex) - 50;
    const letters = mapping[mappingIndex];
    for (const letter of letters) {
      path.push(letter);
      search(digitIndex + 1);
      path.pop();
    }
  };

  search(0);
  return results;
}
```

### 13. Code Walkthrough
The length and character scan finish before result allocation, so an invalid character cannot produce partial output. Because validation restricts input to ASCII `'2'..'9'`, subtracting code point 50 yields mapping indexes zero through seven. Each recursive level corresponds to exactly one digit, and left-to-right letter iteration fixes Cartesian-product order. The reusable path is joined only at a leaf, creating a fresh string for every result. Empty input follows its explicit `[]` contract rather than treating the empty product as `[""]`.

### 14. Complexity Analysis
Let `P` be the product of the mapping sizes for the supplied digits, so `P <= 4^n`. Every output string has length `n`; joining each path therefore takes `Theta(P * n)` time and the output occupies `Theta(P * n)` space. The path and recursion stack use `O(n)` auxiliary space excluding output. Validation takes `O(n)` time.

### 15. Edge Cases
- Empty input returns `[]`.
- A single digit returns its three or four letters in mapping order.
- Digits `7` and `9` each contribute four choices.
- Characters `0`, `1`, whitespace, signs, and non-ASCII numeral lookalikes throw.
- Eleven valid digits throw before output allocation.
- The input string is immutable, and each returned string is a fresh leaf value.

### 16. Common Mistakes
- Returning `[""]` for empty input despite the stated contract.
- Accepting `0` or `1` with an empty mapping.
- Using `parseInt`, which can obscure the ASCII-only requirement.
- Producing output before validating the entire string.
- Reversing loop order and changing deterministic output.
- Calling the algorithm `O(4^n)` while ignoring length-`n` string materialization.

### 17. Interview Follow-ups
1. Stream combinations lazily. 2. Stop after the first `k` outputs. 3. Add a dictionary-prefix filter. 4. Support a caller-provided ordered mapping.

### 18. Variations
- T9 dictionary search. - Cartesian products of arrays. - Case toggling. - Symbol expansion. - Mnemonic generation.

### 19. Pattern Connections
This search is a full decision tree with fixed depth and no invalid internal states. It is the cleanest form of choose-recurse-unchoose and a useful contrast with combination sum, where start indexes and remaining values constrain which children exist.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/phone-combinations` with `{ digits: unknown }`. Require a string of at most 10 ASCII characters from `'2'` through `'9'`, then call only `letterCombinations`; map `RangeError` to HTTP `400`. Return combinations plus at most 200 optimized events such as `{ digitIndex, letterIndex, depth, action: "choose" | "unchoose" | "emit" }` and `truncated`. Do not include the path or accumulated result array in each event. Apply response-byte limits because ten valid digits can produce more than half a million strings.

### 21. Mini Challenge
Add a positive result limit that stops the DFS after exactly that many combinations without changing the prefix of the normal deterministic output order.

## Problem 142 — Generate Parentheses

### 1. Problem Statement
Given an integer `n` from 0 through 10, return every well-formed string containing exactly `n` pairs of parentheses. Use deterministic depth-first order that tries `'('` before `')'`. Return `[""]` when `n = 0`, return fresh strings, and throw `RangeError` for nonintegers or values outside the allowed range.

### 2. Example
```text
Input:  n = 3
Output: ["((()))", "(()())", "(())()", "()(())", "()()()"]

Input:  n = 0
Output: [""]
```

### 3. Understand the Problem
A prefix is valid only when it has used at most `n` opening parentheses and never has more closing than opening parentheses. Once a valid prefix reaches length `2n`, those two conditions guarantee a complete well-formed string.

#### In Simple Terms
Add an opening parenthesis while any remain, and add a closing parenthesis only when there is an unmatched opening parenthesis to close.

### 4. Real-World Analogy
Opening a parenthesis is like opening a nested box, and closing one seals the most recently opened box. You may open at most `n` boxes and can never close a box that has not been opened.

### 5. Key Observation
Tracking counts is enough; the exact stack contents are unnecessary because there is only one bracket type. The conditions `open < n` and `close < open` prevent every impossible prefix before recursion, while trying open first establishes the required output order.

### 6. Pattern
```text
Pattern: Constraint-Pruned Backtracking
Why this pattern applies: validity can be decided from prefix counts
How to recognize it: partial choices have invariants that reject entire subtrees
Similar problems: balanced brackets, Dyck words, constrained binary strings
```

### 7. Brute Force
Validate `n`, then enumerate all `2^(2n)` length-`2n` strings by trying `'('` before `')'` at every position without prefix pruning. At each complete candidate, scan from left to right: increment balance for `'('`, decrement for `')'`, reject if balance becomes negative, and accept only if final balance is zero.

#### Pseudocode
```text
validate integer n from 0 through 10
enumerate every length-2n string over '(' then ')':
    at a complete string:
        scan its balance from left to right
        reject if balance ever becomes negative
        append a fresh string only if final balance is zero
return results
```

#### Brute-Force TypeScript
```typescript
function generateParenthesesBrute(n: number): string[] {
  if (!Number.isInteger(n) || n < 0 || n > 10) {
    throw new RangeError("n must be an integer from 0 through 10");
  }

  const results: string[] = [];
  const path: string[] = [];
  const enumerate = (): void => {
    if (path.length === 2 * n) {
      let balance = 0;
      for (const character of path) {
        balance += character === "(" ? 1 : -1;
        if (balance < 0) return;
      }
      if (balance === 0) results.push(path.join(""));
      return;
    }

    path.push("(");
    enumerate();
    path.pop();
    path.push(")");
    enumerate();
    path.pop();
  };

  enumerate();
  return results;
}
```

#### Visualization
```text
n = 2, all depth-4 leaves
((((), (((), (()(), (()), )(((), )(()), ))((), ))))
valid leaves after balance scan: (()), ()()
```

#### Dry Run
For `n = 2`, enumeration creates 16 complete strings. The leaf `(())` never has negative balance and ends at zero, so it is emitted. The leaf `())(` becomes negative at its third character and is rejected. Because `'('` is always explored first, `(())` appears before `()()`.

#### Brute-Force Complexity
There are `2^(2n) = 4^n` complete candidates, and validating or materializing a candidate costs `O(n)`, for `O(n * 4^n)` time. The path and recursion stack use `O(n)` auxiliary space excluding output. The valid fresh strings use `Theta(C_n * n)` output space, where `C_n` is the `n`th Catalan number.

### 8. Why Brute Force Is Not Ideal
Most binary strings either close before an opening exists or use too many parentheses of one kind. Validity of these prefixes is already known, so generating their descendants wastes exponential work. Count-constrained backtracking removes an entire invalid subtree as soon as either rule would be violated.

### 9. Optimal Approach
Keep `open` and `close` counts plus a reusable character path. Try `'('` first when `open < n`. Try `')'` second only when `close < open`. When the path reaches length `2n`, join it into a fresh string. For `n = 0`, the initial path is already complete and emits the empty string.

### 10. Visualization
```text
n = 3
[]
+-- ( -> (( -> ((( -> ((()))
|         `-- (() -> (()()), (())()
`-- after completing the '(' branch, prefixes beginning ')' never exist
    later valid '(' ')' branch -> ()(()), ()()()
```

### 11. Step-by-Step Dry Run
| Event | Open | Close | Path | Result |
|---|---:|---:|---|---|
| choose `(` | 1 | 0 | `(` | - |
| choose `(` | 2 | 0 | `((` | - |
| choose `(` | 3 | 0 | `(((` | only closes remain |
| choose three `)` | 3 | 3 | `((()))` | emit fresh string |
| backtrack | 2 | 1 | `(()` | next valid leaf is `(()())` |

### 12. TypeScript Solution
```typescript
function generateParentheses(n: number): string[] {
  if (!Number.isInteger(n) || n < 0 || n > 10) {
    throw new RangeError("n must be an integer from 0 through 10");
  }

  const results: string[] = [];
  const path: string[] = [];
  const search = (open: number, close: number): void => {
    if (path.length === 2 * n) {
      results.push(path.join(""));
      return;
    }

    if (open < n) {
      path.push("(");
      search(open + 1, close);
      path.pop();
    }
    if (close < open) {
      path.push(")");
      search(open, close + 1);
      path.pop();
    }
  };

  search(0, 0);
  return results;
}
```

### 13. Code Walkthrough
Validation accepts only integers in the documented range. The first branch adds an opening parenthesis only while fewer than `n` have been used. The second adds a closing parenthesis only when an unmatched opening exists. Therefore every visited prefix can still become valid, and a length-`2n` path is automatically balanced. Joining the local path creates an immutable fresh result before backtracking mutates the array. Open-first traversal yields the documented five-string order for `n = 3`; the initial leaf yields `[""]` for zero.

### 14. Complexity Analysis
The algorithm returns `C_n = (1 / (n + 1)) * binomial(2n, n)` strings. Materializing each length-`2n` string requires `Theta(n)` work, so output time and space are `Theta(C_n * n)`. The number of visited valid-prefix search nodes is Catalan-related and within the same output-sensitive asymptotic bound. The mutable path and recursion stack use `O(n)` auxiliary space excluding output.

### 15. Edge Cases
- `n = 0` returns `[""]`.
- `n = 1` returns `["()"]`.
- `n = 10` is accepted and returns 16,796 strings.
- Negative, fractional, `NaN`, infinite, or greater-than-10 inputs throw.
- No generated prefix begins with `')'` or has more closes than opens.
- Every emitted string is a fresh immutable value.

### 16. Common Mistakes
- Returning `[]` instead of `[""]` for zero pairs.
- Allowing `close <= open`, which can add a close when counts are equal.
- Stopping after `n` characters instead of `2n`.
- Trying `')'` first and changing the required deterministic order.
- Revalidating every optimized leaf instead of maintaining prefix invariants.
- Reporting `O(2^n)` rather than Catalan output cost or brute-force `O(n * 4^n)`.

### 17. Interview Follow-ups
1. Count valid strings without listing them. 2. Generate multiple bracket types. 3. Return the `k`th string in this order. 4. Stream results with cancellation.

### 18. Variations
- Balanced braces. - Dyck paths. - Remove invalid parentheses. - Minimum insertions for balance. - Parenthesis ranking and unranking.

### 19. Pattern Connections
The `close <= open` prefix invariant is the same nonnegative-height rule used by Dyck paths, while the final equality corresponds to returning to height zero. This is backtracking as implicit generation of Catalan objects rather than filtering all binary strings.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/generate-parentheses` with `{ n: unknown }`. Require an integer from 0 through 10, then call only `generateParentheses`; map `RangeError` to HTTP `400`. Return strings plus at most 200 optimized events such as `{ open, close, depth, action: "choose-open" | "choose-close" | "unchoose" | "emit" }` and `truncated`. Do not repeat the complete path or accumulated output in each event. Enforce response limits even though the input is small because `n = 10` returns 16,796 strings.

### 21. Mini Challenge
Return only strings with maximum nesting depth at most `k`, preserving open-before-close order and the `[""]` result when `n = 0`.

## Problem 143 — Word Search

### 1. Problem Statement
Given a mutable `string[][]` board and a string `word`, determine whether the word can be formed from a path of orthogonally adjacent cells. A cell may be used at most once in one path. Match exact Unicode code points without normalization or case folding, and restore the board exactly before returning.

The board may be empty. If it is nonempty, it must have 1 through 6 rows, every row must have the same 1 through 6 columns, and every cell must be a string containing exactly one Unicode code point. The word may contain at most 20 Unicode code points. Validate the entire board and word before search. An empty word returns `true`, including on an empty board; a nonempty word on an empty board returns `false`. Invalid dimensions, cells, or word length throw `RangeError` before any mutation.

### 2. Example
```text
Input:
board = [
  ["A", "B", "C", "E"],
  ["S", "F", "C", "S"],
  ["A", "D", "E", "E"]
]
word = "ABCCED"

Output: true
Board after return: exactly equal to board before the call
```

### 3. Understand the Problem
The path can start at any cell. After matching one code point, it may continue up, right, down, or left, but not outside the board and not through a cell already used by that path. Different candidate paths may reuse the same cells because each failed choice is undone before the next choice begins.

#### In Simple Terms
Find the first letter anywhere, then walk one square at a time without moving diagonally or stepping on the same square twice.

### 4. Real-World Analogy
Imagine tracing a word through letter tiles with a washable marker. Each tile can be marked once during one attempt. When an attempt finishes, erase every mark so the board is exactly ready for the next attempt.

### 5. Key Observation
At recursion depth `i`, only cells matching code point `wordPoints[i]` can continue the path. Marking the chosen cell unavailable prevents reuse. A mutable clone lets the search use an unambiguous `null` marker even though any string code point is legal, while `try/finally` guarantees restoration on both successful and failed returns. The caller's board is never mutated.

### 6. Pattern
```text
Pattern: Grid Backtracking
Why this pattern applies: each matching cell creates up to four constrained next moves
How to recognize it: local adjacency, per-path visited state, and choose-recurse-restore
Similar problems: maze paths, island paths, Boggle, grid cycle detection
```

### 7. Brute Force
Validate the complete input first. For every board cell, enumerate every length-`L - 1` sequence of the four directions, where `L` is the word length. Do not prune partial sequences. At each complete direction sequence, replay the proposed path from its start, checking bounds, repeated coordinates, and exact code-point equality. Use a `Set<number>` with key `row * columns + column` to reject cell reuse. This version never mutates the board.

#### Pseudocode
```text
validate all dimensions, all cells, and the word length
if word is empty: return true
if board is empty: return false
for every start cell:
    enumerate every sequence of L - 1 choices from [up, right, down, left]
    for each complete sequence:
        replay it from the start
        reject on mismatch, out-of-bounds movement, or a repeated coordinate
        return true if all L cells match
return false
```

#### Brute-Force TypeScript
```typescript
function wordExistsBrute(board: string[][], word: string): boolean {
  const wordPoints = [...word];
  if (wordPoints.length > 20) {
    throw new RangeError("word must contain at most 20 Unicode code points");
  }
  if (board.length > 6) {
    throw new RangeError("board must contain at most 6 rows");
  }

  const columns = board.length === 0 ? 0 : board[0].length;
  if (board.length > 0 && (columns === 0 || columns > 6)) {
    throw new RangeError("a nonempty board must contain 1 through 6 columns");
  }
  for (const row of board) {
    if (row.length !== columns) {
      throw new RangeError("board must be rectangular");
    }
    for (const cell of row) {
      if (typeof cell !== "string" || [...cell].length !== 1) {
        throw new RangeError("every cell must contain exactly one Unicode code point");
      }
    }
  }

  if (wordPoints.length === 0) return true;
  if (board.length === 0) return false;

  const directions: ReadonlyArray<readonly [number, number]> = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];
  const choices: number[] = [];
  const matchesCompletePath = (startRow: number, startColumn: number): boolean => {
    let row = startRow;
    let column = startColumn;
    const used = new Set<number>();
    if (board[row][column] !== wordPoints[0]) return false;
    used.add(row * columns + column);

    for (let index = 1; index < wordPoints.length; index += 1) {
      const [rowChange, columnChange] = directions[choices[index - 1]];
      row += rowChange;
      column += columnChange;
      if (row < 0 || row >= board.length || column < 0 || column >= columns) {
        return false;
      }
      const key = row * columns + column;
      if (used.has(key) || board[row][column] !== wordPoints[index]) return false;
      used.add(key);
    }
    return true;
  };

  const enumerateDirections = (startRow: number, startColumn: number): boolean => {
    if (choices.length === wordPoints.length - 1) {
      return matchesCompletePath(startRow, startColumn);
    }
    for (let direction = 0; direction < directions.length; direction += 1) {
      choices.push(direction);
      if (enumerateDirections(startRow, startColumn)) return true;
      choices.pop();
    }
    return false;
  };

  for (let row = 0; row < board.length; row += 1) {
    for (let column = 0; column < columns; column += 1) {
      if (enumerateDirections(row, column)) return true;
    }
  }
  return false;
}
```

#### Visualization
```text
word length L = 4, one start cell
direction sequences: UUU, UUR, UUD, UUL, URU, ... , LLL
only after reaching a sequence leaf is the proposed path replayed and checked
```

#### Dry Run
For word `ABCCED`, the brute-force search starts at each of 12 cells and enumerates direction sequences of length five. From the top-left `A`, the sequence right, right, down, down, left replays coordinates `(0,0)`, `(0,1)`, `(0,2)`, `(1,2)`, `(2,2)`, `(2,1)`. They are distinct, in bounds, and spell the word, so the function returns `true` without changing the board.

#### Brute-Force Complexity
Let `R` and `C` be the board dimensions and `L` the word length. There are `R * C` starts and `4^(L - 1)` direction sequences; replaying one sequence costs `O(L)`. The time bound is therefore `O(R * C * 4^L * L)`. The choices array and replay set use `O(L)` auxiliary space. The board is not mutated, and no output collection is allocated.

### 8. Why Brute Force Is Not Ideal
Most direction sequences leave the board, revisit a cell, or mismatch the next code point near the beginning. Waiting until a complete sequence exists before checking it repeats those known-invalid prefixes exponentially many times. Backtracking can reject each bad prefix immediately.

### 9. Optimal Approach
Validate the complete input before cloning or searching. Convert the word to code points with spread iteration. Reject immediately when the word is longer than the number of cells or requires more copies of a code point than the board contains. Clone the board as `Array<Array<string | null>>`, preserving every caller cell. From each matching start, run DFS in fixed up-right-down-left order. Save the current value, replace it with `null`, explore neighbors, and restore the saved value in `finally`, even when a successful branch returns early.

### 10. Visualization
```text
ABCCED on the example board
(0,0) A -> (0,1) B -> (0,2) C -> (1,2) C -> (2,2) E -> (2,1) D
   null       null       null       null       null       null

unwind: every null is restored to its exact saved string
caller board: never changed because DFS works on a clone
```

### 11. Step-by-Step Dry Run
| Event | Coordinate | Needed | Working cell after event | Path result |
|---|---|---|---|---|
| choose | `(0,0)` | `A` | `null` | continue |
| choose right | `(0,1)` | `B` | `null` | continue |
| choose right | `(0,2)` | `C` | `null` | continue |
| choose down | `(1,2)` | `C` | `null` | continue |
| choose down, then left | `(2,2)`, `(2,1)` | `E`, `D` | temporarily `null` | success |
| unwind through `finally` | all six cells | - | exact saved strings restored | `true` |

### 12. TypeScript Solution
```typescript
function wordExists(board: string[][], word: string): boolean {
  const wordPoints = [...word];
  if (wordPoints.length > 20) {
    throw new RangeError("word must contain at most 20 Unicode code points");
  }
  if (board.length > 6) {
    throw new RangeError("board must contain at most 6 rows");
  }

  const columns = board.length === 0 ? 0 : board[0].length;
  if (board.length > 0 && (columns === 0 || columns > 6)) {
    throw new RangeError("a nonempty board must contain 1 through 6 columns");
  }
  const available = new Map<string, number>();
  for (const row of board) {
    if (row.length !== columns) {
      throw new RangeError("board must be rectangular");
    }
    for (const cell of row) {
      if (typeof cell !== "string" || [...cell].length !== 1) {
        throw new RangeError("every cell must contain exactly one Unicode code point");
      }
      available.set(cell, (available.get(cell) ?? 0) + 1);
    }
  }

  if (wordPoints.length === 0) return true;
  if (board.length === 0 || wordPoints.length > board.length * columns) return false;
  const required = new Map<string, number>();
  for (const point of wordPoints) {
    const count = (required.get(point) ?? 0) + 1;
    if (count > (available.get(point) ?? 0)) return false;
    required.set(point, count);
  }

  const working: Array<Array<string | null>> = board.map((row) => [...row]);
  const directions: ReadonlyArray<readonly [number, number]> = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];
  const search = (row: number, column: number, wordIndex: number): boolean => {
    if (row < 0 || row >= working.length || column < 0 || column >= columns) {
      return false;
    }
    const saved = working[row][column];
    if (saved !== wordPoints[wordIndex]) return false;

    working[row][column] = null;
    try {
      if (wordIndex === wordPoints.length - 1) return true;
      for (const [rowChange, columnChange] of directions) {
        if (search(row + rowChange, column + columnChange, wordIndex + 1)) {
          return true;
        }
      }
      return false;
    } finally {
      working[row][column] = saved;
    }
  };

  for (let row = 0; row < working.length; row += 1) {
    for (let column = 0; column < columns; column += 1) {
      if (search(row, column, 0)) return true;
    }
  }
  return false;
}
```

### 13. Code Walkthrough
Spread iteration defines both the word length and each legal cell in Unicode code points, so surrogate pairs such as an emoji count once; no normalization is performed. Validation scans every row and cell before the clone is created. Frequency checks reject impossible words without search. DFS checks bounds before indexing, compares the needed code point, stores the exact cell value, marks the clone with `null`, and restores in `finally`. Thus restoration occurs on failure, normal completion, and early success, while the caller's mutable board remains byte-for-byte untouched.

### 14. Complexity Analysis
In the worst case, each of `R * C` starts explores up to four choices per word position, giving `O(R * C * 4^L)` time; after the first move, the visited mark reduces the practical branching factor to at most three. Validation, cloning, and frequency pruning take `O(R * C + L)` time. The working clone uses `O(R * C)` space, and the recursion stack uses `O(L)` auxiliary space. The boolean output is constant space.

### 15. Edge Cases
- `word = ""` returns `true` after full validation, even when `board = []`.
- A nonempty word with `board = []` returns `false`.
- A nonempty board with an empty row, ragged rows, more than 6 rows or columns, or a cell containing zero or multiple code points throws.
- A 21-code-point word throws before search or mutation.
- Diagonal-only matches and paths that reuse one coordinate return `false`.
- Canonically equivalent but differently encoded strings do not match because there is no normalization.
- The caller board and the internal working clone are exactly restored for successful and failed searches.

### 16. Common Mistakes
- Treating UTF-16 code units as code points by indexing the word directly.
- Accepting ragged rows or multi-code-point cell strings.
- Using a string sentinel that could equal a legal cell.
- Returning early before unmarking a successful path.
- Checking only dimensions and mutating before validating every cell.
- Allowing diagonal movement or reusing a cell within one path.
- Claiming constant space while storing a board clone and recursion stack.

### 17. Interview Follow-ups
1. Return one matching coordinate path. 2. Find many dictionary words with a trie. 3. Support diagonal movement. 4. Normalize both cells and the word under an explicit Unicode policy.

### 18. Variations
- Boggle search. - Trie-guided multiword search. - Maze path existence. - Longest increasing grid path. - Grid paths with reusable cells.

### 19. Pattern Connections
The `null` mark is the grid equivalent of a `used` set in permutation search: it records a choice that is unavailable only on the current recursion path. The `finally` block is the unchoose step, making restoration an enforced control-flow property instead of a convention that early returns can bypass.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/word-search` with `{ board: unknown, word: unknown }`. A DTO or pipe must require a rectangular mutable array with at most 6 rows and 6 columns, exact one-code-point string cells, and a string of at most 20 code points; the algorithm must still enforce the same contract. Call only `wordExists`, map `RangeError` to HTTP `400`, and snapshot the validated board before invocation so an integration assertion can verify exact restoration afterward. Return only `{ exists }` plus at most 200 optimized events containing coordinates, word index, and action, never a copy of the board per event.

### 21. Mini Challenge
Return the first coordinate path in the same start-cell and up-right-down-left order, using fresh coordinate pairs and still guaranteeing exact board restoration.

## Problem 144 — N-Queens

### 1. Problem Statement
Given an integer `n` from 0 through 10, return every placement of `n` queens on an `n x n` chessboard such that no two queens share a row, column, or diagonal. Represent each solution as `string[]`, using `'Q'` for a queen and `'.'` for an empty square, so the complete return type is `string[][]`. Search rows from top to bottom and columns from left to right. Return `[[]]` when `n = 0`, return fresh board arrays with no shared mutable aliases, and throw `RangeError` for nonintegers or values outside the allowed range.

### 2. Example
```text
Input: n = 4
Output: [
  [".Q..", "...Q", "Q...", "..Q."],
  ["..Q.", "Q...", "...Q", ".Q.."]
]

Input: n = 0
Output: [[]]
```

### 3. Understand the Problem
Exactly one queen is placed in each row. Once a column is chosen for every row, the placement is valid only if all columns, all descending diagonals identified by `row - column`, and all ascending diagonals identified by `row + column` are unique.

#### In Simple Terms
Place one queen on each row, but never put the next queen directly above or diagonally above an earlier queen.

### 4. Real-World Analogy
Think of scheduling one transmitter per floor of a building. Two transmitters interfere if they use the same vertical shaft or either 45-degree line of sight, so each new floor must choose a position outside all three occupied line families.

### 5. Key Observation
Row-by-row placement handles row uniqueness automatically. Three sets are sufficient to test every remaining attack in constant average time. Storing only the selected column per row avoids a mutable `n x n` working board; complete boards are materialized as fresh arrays only at valid leaves.

### 6. Pattern
```text
Pattern: Constraint-Pruned Permutation Backtracking
Why this pattern applies: each row chooses one unused, nonattacking column
How to recognize it: fixed-depth placement with reversible column and diagonal constraints
Similar problems: Sudoku, graph coloring, rook placement, exact cover
```

### 7. Brute Force
Validate `n`, then generate all `n^n` assignments that choose one of `n` columns independently for each row. Do not prune repeated columns or diagonal conflicts while building an assignment. At each complete assignment, scan its `n` positions with three sets. If every column and diagonal key is unique, materialize a fresh board. For `n = 0`, the one empty assignment is valid and emits one fresh empty board.

#### Pseudocode
```text
validate integer n from 0 through 10
enumerate every length-n sequence whose values are columns 0 through n - 1:
    at a complete assignment:
        create empty column and diagonal sets
        reject if any position repeats a column, row-column, or row+column key
        otherwise map every position to a fresh board row and append that board
return all boards
```

#### Brute-Force TypeScript
```typescript
function solveNQueensBrute(n: number): string[][] {
  if (!Number.isInteger(n) || n < 0 || n > 10) {
    throw new RangeError("n must be an integer from 0 through 10");
  }

  const results: string[][] = [];
  const positions: number[] = [];
  const enumerate = (): void => {
    if (positions.length === n) {
      const columns = new Set<number>();
      const descending = new Set<number>();
      const ascending = new Set<number>();
      for (let row = 0; row < n; row += 1) {
        const column = positions[row];
        if (
          columns.has(column) ||
          descending.has(row - column) ||
          ascending.has(row + column)
        ) {
          return;
        }
        columns.add(column);
        descending.add(row - column);
        ascending.add(row + column);
      }

      const board = positions.map(
        (column) => ".".repeat(column) + "Q" + ".".repeat(n - column - 1),
      );
      results.push(board);
      return;
    }

    for (let column = 0; column < n; column += 1) {
      positions.push(column);
      enumerate();
      positions.pop();
    }
  };

  enumerate();
  return results;
}
```

#### Visualization
```text
n = 3 brute-force assignments
[0,0,0], [0,0,1], [0,0,2], ... , [2,2,2]
3^3 complete assignments are generated; all are rejected at their leaves
```

#### Dry Run
For `n = 2`, enumeration generates `[0,0]`, `[0,1]`, `[1,0]`, and `[1,1]`. The first and last repeat a column. In `[0,1]`, both queens share descending key `0`; in `[1,0]`, both share ascending key `1`. Every leaf is rejected, so the result is `[]`.

#### Brute-Force Complexity
There are `n^n` complete assignments, and validating one takes `O(n)`, for `O(n^(n + 1))` search time. Materializing the `S` valid boards additionally costs `O(S * n^2)` time and output space. The assignment, validation sets, and recursion stack use `O(n)` auxiliary space excluding output. The input bound makes the function finite, although the brute-force version is intentionally impractical near `n = 10`.

### 8. Why Brute Force Is Not Ideal
Once two chosen queens attack each other, adding more rows can never repair that conflict. Brute force still generates every descendant of the invalid prefix. Maintaining occupied columns and diagonals during construction rejects the entire subtree at the first conflict.

### 9. Optimal Approach
Validate before allocating output. Recurse by row, trying columns from `0` through `n - 1`. Skip a column when it is already in the column set, when `row - column` is in the descending-diagonal set, or when `row + column` is in the ascending-diagonal set. Otherwise add all three keys and the column position, recurse, then remove them. At row `n`, map the positions to a newly allocated board and append it. The empty initial position emits `[[]]` for `n = 0`.

### 10. Visualization
```text
n = 4, first solution positions [1, 3, 0, 2]
row 0: .Q..
row 1: ...Q
row 2: Q...
row 3: ..Q.

occupied keys per queen: column, row-column, row+column
```

### 11. Step-by-Step Dry Run
| Row | Tried column | Decision | Positions |
|---:|---:|---|---|
| 0 | 0 | choose | `[0]` |
| 1 | 0, 1 | reject column/diagonal | `[0]` |
| 1 | 2 | choose, later dead end | `[0, 2]` |
| 0 | 1 | choose after backtracking | `[1]` |
| 1, 2, 3 | 3, 0, 2 | choose each | `[1, 3, 0, 2]` |
| 4 | - | emit fresh first board | unchanged working positions |

### 12. TypeScript Solution
```typescript
function solveNQueens(n: number): string[][] {
  if (!Number.isInteger(n) || n < 0 || n > 10) {
    throw new RangeError("n must be an integer from 0 through 10");
  }

  const results: string[][] = [];
  const positions: number[] = [];
  const columns = new Set<number>();
  const descending = new Set<number>();
  const ascending = new Set<number>();
  const search = (row: number): void => {
    if (row === n) {
      const board = positions.map(
        (column) => ".".repeat(column) + "Q" + ".".repeat(n - column - 1),
      );
      results.push(board);
      return;
    }

    for (let column = 0; column < n; column += 1) {
      const descendingKey = row - column;
      const ascendingKey = row + column;
      if (
        columns.has(column) ||
        descending.has(descendingKey) ||
        ascending.has(ascendingKey)
      ) {
        continue;
      }

      positions.push(column);
      columns.add(column);
      descending.add(descendingKey);
      ascending.add(ascendingKey);
      search(row + 1);
      ascending.delete(ascendingKey);
      descending.delete(descendingKey);
      columns.delete(column);
      positions.pop();
    }
  };

  search(0);
  return results;
}
```

### 13. Code Walkthrough
Integer validation happens before result allocation. The recursion depth is the row, so two queens can never share a row. The three sets reject all column and diagonal attacks before recursion. Columns are tried left to right, fixing deterministic solution order. At a leaf, `positions.map` allocates a new outer board array and creates every immutable row string; no returned board aliases the mutable positions path or another returned board. Every chosen key is deleted during backtracking before the next column is tried.

### 14. Complexity Analysis
Column uniqueness limits the search to prefixes of permutations, giving the common `O(n!)` search-node upper bound. Materializing `S` solutions costs `Theta(S * n^2)` time and output space because each board contains `n^2` characters. Thus total time is `O(n! + S * n^2)`. The positions array, recursion stack, and three sets use `O(n)` auxiliary space excluding output; no `n x n` working board is stored.

### 15. Edge Cases
- `n = 0` returns `[[]]`, the single empty-board solution.
- `n = 1` returns `[["Q"]]`.
- `n = 2` and `n = 3` return `[]`.
- `n = 4` returns the two boards in the documented order.
- Negative, fractional, `NaN`, infinite, or greater-than-10 inputs throw before output allocation.
- Every solution is a fresh board array, and row strings are immutable; mutating one returned board array cannot change another.

### 16. Common Mistakes
- Returning `[]` rather than `[[]]` for `n = 0`.
- Checking only columns and one diagonal direction.
- Using `Math.abs(row - column)` as a diagonal key and merging distinct diagonals.
- Sharing one mutable board array across results.
- Forgetting to delete one constraint during backtracking.
- Trying columns in nondeterministic set order and changing output order.
- Omitting the `S * n^2` cost of materializing returned boards.

### 17. Interview Follow-ups
1. Return only the number of solutions. 2. Stop after the first solution. 3. Use bit masks instead of sets. 4. Place queens on a board with blocked squares.

### 18. Variations
- N-Rooks. - Sudoku. - Exact cover. - Graph coloring. - Maximum nonattacking queens with obstacles.

### 19. Pattern Connections
N-Queens is permutation search plus diagonal constraints. Column uniqueness turns row choices into a partial permutation, while diagonal sets add problem-specific pruning. Materializing only at leaves is the same output-copy discipline used by permutations and subsets: working state is reused, returned state is fresh.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/n-queens` with `{ n: unknown }`. Require an integer from 0 through 10, call only `solveNQueens`, and map `RangeError` to HTTP `400`. Return boards plus at most 200 optimized events such as `{ row, column, action: "choose" | "reject" | "unchoose" | "emit" }` and `truncated`; never attach a full board to every event. Apply response-byte and execution limits because result materialization grows rapidly. In an integration test, mutate one returned board array and assert every other solution remains unchanged to verify output-copy independence.

### 21. Mini Challenge
Implement a counting-only bit-mask version for `n` from 0 through 15 while preserving the convention that the empty board has one solution.

## Problem 145 — Sudoku Solver

### 1. Problem Statement
Given a mutable `string[][]` Sudoku board, solve it in place and return whether a solution exists. The board must be exactly `9 x 9`, and every entry must be `'.'` or a digit from `'1'` through `'9'`. Before any mutation, validate the complete shape, every symbol, and the initial consistency: no given digit may appear twice in one row, column, or `3 x 3` box. Invalid input throws `RangeError` before mutation.

Try candidate digits deterministically from `'1'` through `'9'`. If the board is solvable, return `true` and leave one valid solution in the caller's board. A valid full board returns `true` unchanged. If the board is unsolvable, return `false` and restore every cell exactly to its initial value. Only one solution is required; uniqueness is not required.

### 2. Example
```text
Input:
[
  ["5","3",".",".","7",".",".",".","."],
  ["6",".",".","1","9","5",".",".","."],
  [".","9","8",".",".",".",".","6","."],
  ["8",".",".",".","6",".",".",".","3"],
  ["4",".",".","8",".","3",".",".","1"],
  ["7",".",".",".","2",".",".",".","6"],
  [".","6",".",".",".",".","2","8","."],
  [".",".",".","4","1","9",".",".","5"],
  [".",".",".",".","8",".",".","7","9"]
]

Output: true
Board after return: one valid completed Sudoku grid
```

### 3. Understand the Problem
Each empty cell needs one digit that does not duplicate a digit already present in its row, column, or box. A tentative digit can make a later cell impossible, so the solver must undo that digit and try the next candidate. Given cells are fixed and must never be changed.

#### In Simple Terms
Fill each dot with a legal digit. When a choice leads to a dead end, erase only the digits you added and try another choice.

### 4. Real-World Analogy
Imagine completing a logic puzzle in pencil. Before writing, check the three relevant groups. If later deductions fail, erase each penciled answer in reverse order while leaving the printed clues untouched.

### 5. Key Observation
A partial assignment is legal exactly when its chosen digit is absent from three groups: one row, one column, and one box. Backtracking preserves the public mutation contract when every failed assignment is reset to `'.'`. Because validation performs no writes and DFS changes only original empty cells, exhausting the search restores an unsolvable board exactly.

### 6. Pattern
```text
Pattern: Constraint-Satisfaction Backtracking
Why this pattern applies: each empty cell chooses one digit under three reversible constraints
How to recognize it: a partial assignment, local legality tests, and dead-end recovery
Similar problems: N-Queens, graph coloring, Latin squares, exact cover
```

### 7. Brute Force
Validate all 81 cells and all initial row, column, and box constraints before writing to the board. DFS repeatedly scans row-major for the first `'.'`. For that cell, try digits `'1'` through `'9'`. For each candidate, scan its complete row, column, and `3 x 3` box. Assign a legal digit, recurse, and reset the cell to `'.'` when the branch fails. If no empty cell remains, return `true`.

#### Pseudocode
```text
validate exact 9 x 9 shape and legal symbols
validate that all given digits are consistent
search:
    find the first empty cell in row-major order
    if none exists: return true
    for digit from '1' through '9':
        scan the cell's row, column, and box
        if digit is legal:
            assign digit
            if search succeeds: return true
            restore '.'
    return false
```

#### Brute-Force TypeScript
```typescript
function solveSudokuBrute(board: string[][]): boolean {
  if (!Array.isArray(board) || board.length !== 9) {
    throw new RangeError("board must contain exactly 9 rows");
  }
  for (const row of board) {
    if (!Array.isArray(row) || row.length !== 9) {
      throw new RangeError("every row must contain exactly 9 cells");
    }
  }

  const rows = Array.from({ length: 9 }, () => new Set<string>());
  const columns = Array.from({ length: 9 }, () => new Set<string>());
  const boxes = Array.from({ length: 9 }, () => new Set<string>());
  for (let row = 0; row < 9; row += 1) {
    for (let column = 0; column < 9; column += 1) {
      const cell = board[row][column];
      if (typeof cell !== "string" || (cell !== "." && !/^[1-9]$/.test(cell))) {
        throw new RangeError("every cell must be '.' or a digit from '1' through '9'");
      }
      if (cell === ".") continue;

      const box = Math.floor(row / 3) * 3 + Math.floor(column / 3);
      if (rows[row].has(cell) || columns[column].has(cell) || boxes[box].has(cell)) {
        throw new RangeError("given digits must be consistent");
      }
      rows[row].add(cell);
      columns[column].add(cell);
      boxes[box].add(cell);
    }
  }

  const isLegal = (row: number, column: number, digit: string): boolean => {
    for (let index = 0; index < 9; index += 1) {
      if (board[row][index] === digit || board[index][column] === digit) {
        return false;
      }
    }

    const boxRow = Math.floor(row / 3) * 3;
    const boxColumn = Math.floor(column / 3) * 3;
    for (let rowOffset = 0; rowOffset < 3; rowOffset += 1) {
      for (let columnOffset = 0; columnOffset < 3; columnOffset += 1) {
        if (board[boxRow + rowOffset][boxColumn + columnOffset] === digit) {
          return false;
        }
      }
    }
    return true;
  };

  const search = (): boolean => {
    let emptyRow = -1;
    let emptyColumn = -1;
    for (let row = 0; row < 9 && emptyRow === -1; row += 1) {
      for (let column = 0; column < 9; column += 1) {
        if (board[row][column] === ".") {
          emptyRow = row;
          emptyColumn = column;
          break;
        }
      }
    }
    if (emptyRow === -1) return true;

    for (let digit = 1; digit <= 9; digit += 1) {
      const symbol = String(digit);
      if (!isLegal(emptyRow, emptyColumn, symbol)) continue;

      board[emptyRow][emptyColumn] = symbol;
      if (search()) return true;
      board[emptyRow][emptyColumn] = ".";
    }
    return false;
  };

  return search();
}
```

#### Visualization
```text
first row-major empty cell
        |
[5, 3, ., ., 7, ., ., ., .]
        |
try 1 -> choose, recurse, eventually reach a dead end
restore '.', try 2
later dead end -> restore '.', try next legal digit
```

#### Dry Run
For the example board, the first empty cell is `(0,2)`. Digit `1` is initially legal and is assigned, but that branch eventually reaches a cell with no legal digit. Recursive failure restores `(0,2)` to `'.'`, and the loop tries `2`, then later candidates in ascending order. A successful branch returns without erasing its solved assignments.

#### Brute-Force Complexity
Let `E` be the number of initial empty cells. The search tries at most nine values at each depth, so the fixed-board worst case is `O(9^E)`. Each candidate legality check scans at most 27 positions, and finding the next empty scans at most 81 cells; both are constants for a `9 x 9` board, yielding the same `O(9^E)` fixed-size bound, or explicitly `O(9^E * 81)`. The recursion stack uses `O(E)` auxiliary space. Mutation is in place, so there is no additional output structure.

### 8. Why Brute Force Is Not Ideal
The first row-major empty cell may have many candidates while another empty cell has only one or none. Repeatedly scanning three groups also rediscovers the same occupancy information. Bit masks make legality constant-time, and minimum-remaining-values ordering exposes forced moves and dead ends earlier.

### 9. Optimal Approach
Validate the entire shape and all symbols while building nine row masks, nine column masks, nine box masks, and an array of empty coordinates. Bits `0` through `8` represent digits `'1'` through `'9'`. Initial duplicate bits throw before any board write.

At recursion position `p`, scan `empties[p...]` and compute each cell's candidate mask. Choose the cell with the fewest candidates, swap it into position `p`, and return `false` immediately if it has zero candidates. Iterate candidates by repeatedly taking the lowest set bit, which preserves digit order. Set the board cell and all three masks, recurse, then clear exactly those changes on failure. Swap the coordinate back when the entire call fails. Internal coordinate order does not affect the caller; on success, solved board assignments intentionally remain.

### 10. Visualization
```text
used bits = rowMask | columnMask | boxMask
candidate bits = 111111111 & ~used

remaining cells: (0,2): 001001011  four candidates
                 (4,4): 000010000  one candidate <- MRV chooses this

lowest candidate bit -> ascending digit order
```

### 11. Step-by-Step Dry Run
| Event | MRV cell | Candidate mask | Action |
|---|---|---|---|
| build | all givens | row/column/box bits | no board mutation |
| choose | cell with one candidate | lowest set bit | assign its digit |
| update | same coordinate | one bit | set three masks |
| recurse | next MRV cell | recomputed from masks | continue |
| dead end | a remaining cell | `000000000` | return `false` |
| unwind | previous coordinate | chosen bit | clear masks and restore `'.'` |
| success | no cells remain | - | return `true`, keep solution |

### 12. TypeScript Solution
```typescript
function solveSudoku(board: string[][]): boolean {
  if (!Array.isArray(board) || board.length !== 9) {
    throw new RangeError("board must contain exactly 9 rows");
  }
  for (const row of board) {
    if (!Array.isArray(row) || row.length !== 9) {
      throw new RangeError("every row must contain exactly 9 cells");
    }
  }

  const rowMasks = Array<number>(9).fill(0);
  const columnMasks = Array<number>(9).fill(0);
  const boxMasks = Array<number>(9).fill(0);
  const empties: Array<[number, number]> = [];
  for (let row = 0; row < 9; row += 1) {
    for (let column = 0; column < 9; column += 1) {
      const cell = board[row][column];
      if (typeof cell !== "string" || (cell !== "." && !/^[1-9]$/.test(cell))) {
        throw new RangeError("every cell must be '.' or a digit from '1' through '9'");
      }
      if (cell === ".") {
        empties.push([row, column]);
        continue;
      }

      const digitIndex = cell.charCodeAt(0) - "1".charCodeAt(0);
      const bit = 1 << digitIndex;
      const box = Math.floor(row / 3) * 3 + Math.floor(column / 3);
      if ((rowMasks[row] & bit) !== 0 ||
          (columnMasks[column] & bit) !== 0 ||
          (boxMasks[box] & bit) !== 0) {
        throw new RangeError("given digits must be consistent");
      }
      rowMasks[row] |= bit;
      columnMasks[column] |= bit;
      boxMasks[box] |= bit;
    }
  }

  const fullMask = (1 << 9) - 1;
  const countBits = (mask: number): number => {
    let count = 0;
    while (mask !== 0) {
      mask &= mask - 1;
      count += 1;
    }
    return count;
  };

  const search = (position: number): boolean => {
    if (position === empties.length) return true;

    let bestIndex = position;
    let bestCandidates = 0;
    let bestCount = 10;
    for (let index = position; index < empties.length; index += 1) {
      const [row, column] = empties[index];
      const box = Math.floor(row / 3) * 3 + Math.floor(column / 3);
      const candidates = fullMask &
        ~(rowMasks[row] | columnMasks[column] | boxMasks[box]);
      const candidateCount = countBits(candidates);
      if (candidateCount < bestCount) {
        bestIndex = index;
        bestCandidates = candidates;
        bestCount = candidateCount;
        if (bestCount === 0) break;
      }
    }
    if (bestCount === 0) return false;

    [empties[position], empties[bestIndex]] = [empties[bestIndex], empties[position]];
    const [row, column] = empties[position];
    const box = Math.floor(row / 3) * 3 + Math.floor(column / 3);
    let candidates = bestCandidates;
    while (candidates !== 0) {
      const bit = candidates & -candidates;
      candidates &= candidates - 1;
      const digitIndex = 31 - Math.clz32(bit);

      board[row][column] = String(digitIndex + 1);
      rowMasks[row] |= bit;
      columnMasks[column] |= bit;
      boxMasks[box] |= bit;
      if (search(position + 1)) return true;

      boxMasks[box] &= ~bit;
      columnMasks[column] &= ~bit;
      rowMasks[row] &= ~bit;
      board[row][column] = ".";
    }

    [empties[position], empties[bestIndex]] = [empties[bestIndex], empties[position]];
    return false;
  };

  return search(0);
}
```

### 13. Code Walkthrough
The first two loops establish the exact `9 x 9` shape before any cell access. The next pass validates every symbol and builds masks and empty coordinates without writing to the board; a repeated given bit throws atomically. `fullMask & ~used` keeps only the low nine legal bits. MRV scans all remaining coordinates because a later cell with zero candidates is more urgent than an earlier cell with one. Swapping changes only the local `empties` array. The low-bit operation visits digits in ascending order, and `31 - Math.clz32(bit)` converts a one-bit mask to its zero-based digit index without floating-point logarithms. Failed branches clear masks and reset the assigned cell. Therefore a failed top-level search restores all original dots, while success deliberately retains the completed board.

### 14. Complexity Analysis
Let `E` be the number of initial empty cells. Both approaches remain exponential in the worst case: MRV and masks improve pruning and per-node work but do not guarantee a better asymptotic search bound than `O(9^E)`. At each visited depth, MRV scans at most `E` remaining cells, so a conservative bound is `O(E * 9^E)` search work; in practice the legal candidate tree is much smaller. Validation and mask construction take `O(81)`, and collecting empties takes `O(E)`. Masks and the empty-coordinate array use `O(E + 81)` storage, while recursion uses `O(E)` stack space. The solved board is the mutated input, not additional output space.

### 15. Edge Cases
- A valid already solved board returns `true` without changing any cell.
- An all-dot board is valid and receives the deterministic first solution found.
- A board with fewer or more than nine rows, or any row of non-nine length, throws before mutation.
- Any symbol other than `'.'` or `'1'` through `'9'` throws before mutation.
- Duplicate givens in a row, column, or box throw before mutation.
- A consistent but unsolvable board returns `false` and is exactly equal to its initial snapshot.
- A puzzle with multiple solutions may produce any solution reached by deterministic MRV tie order and ascending digits; uniqueness is not checked.

### 16. Common Mistakes
- Mutating a cell before all shape, symbol, and duplicate validation is complete.
- Validating rows but forgetting initial column or box duplicates.
- Returning `false` without resetting assignments made by failed ancestors.
- Choosing MRV candidates from stale masks after recursive mutation.
- Using arbitrary set iteration and losing deterministic digit order.
- Claiming MRV changes the guaranteed worst-case exponential bound.
- Copying the board and solving only the copy when the contract requires in-place success.
- Treating an unsolvable search as permission to leave partial assignments behind.

### 17. Interview Follow-ups
1. Count solutions up to two to test uniqueness. 2. Return a fresh solved copy instead of mutating the input. 3. Generalize to `n^2 x n^2` boards. 4. Add constraint propagation for naked singles. 5. Compare with Algorithm X and dancing links.

### 18. Variations
- Sudoku validity only. - Sudoku solution counting. - Latin squares. - KenKen. - Killer Sudoku. - Exact cover.

### 19. Pattern Connections
Sudoku combines N-Queens-style constraint maintenance with Word Search-style restoration. Row, column, and box masks are reversible summaries of the current path. MRV is a search-order heuristic, not a correctness condition: it changes which legal variable is assigned next while choose, recurse, and unchoose preserve the same solution set and atomic failure contract.

### 20. NestJS Project Connection
Expose `POST /api/dsa/backtracking/sudoku` with `{ board: unknown }`. A DTO or pipe should require exactly nine arrays of nine legal string symbols, but `solveSudoku` must still enforce the complete contract. Create a request-scoped deep copy with `board.map((row) => [...row])` so concurrent requests and caller-owned DTO data never share a mutable board. Call only the optimized solver. Map `RangeError` to HTTP `400`; if it returns `false`, return HTTP `422` without a partial board; on success, return the solved copy. Emit at most 200 optimized trace events such as `{ row, column, digit, action: "choose" | "unchoose" | "dead-end" }` plus `truncated`, and never include a full board in each event.

### 21. Mini Challenge
Determine whether the initial board has exactly one solution by searching until two solutions are found, while restoring the caller's board exactly regardless of the result.

# Topic Summary

## What You Learned
Recursion needs a reachable base case, a strictly shrinking measure, and a contract that bounds unsafe depth. The call stack stores only the active path, even when the full recursion tree is exponential. Memoization removes repeated states when the state key is complete; it does not help a tree whose states do not overlap.

Backtracking reuses local state through choose, explore, and unchoose. Correctness depends on exact restoration before every sibling and on fresh snapshots for emitted results. Pruning must be justified, deterministic choice order is part of an observable contract, and exponential generators require output-sensitive analysis rather than a promise that recursion makes them cheap.

JavaScript details matter: exact large arithmetic requires `bigint`; JSON endpoints must serialize it as text; string contracts must distinguish UTF-16 code units, Unicode code points, and grapheme clusters; and finite stacks require explicit bounds. Production-facing recursion also needs atomic validation, bounded traces, response limits, and clear ownership of mutable inputs.

## Patterns You Should Recognize
- **Linear descent and unwind:** one smaller call, with work performed before or after it.
- **Two-pointer recursion:** move matching boundaries inward until they meet.
- **Overlapping subproblems:** cache each complete state once, as in Fibonacci.
- **Binary include/exclude trees:** subsequences and subsets branch on each position.
- **Used-choice permutation search:** choose one unused index at each depth.
- **Target-constrained combination search:** advance a start index and prune impossible sums.
- **Prefix-constrained generation:** generate parentheses only while counts remain legal.
- **Grid backtracking:** mark a coordinate, explore neighbors, and restore it exactly.
- **Constraint-satisfaction search:** maintain reversible row, column, diagonal, or box state.
- **MRV ordering:** select the remaining variable with the fewest legal values to expose dead ends early.

## Complexity Cheat Sheet
| Problem | Main approach | Time | Auxiliary space | Required output space |
|---:|---|---|---|---|
| 131 | Recursive countdown and unwind | `O(n)` | `O(n)` | `O(n)` |
| 132 | Recursive factorial | `O(n)` bigint multiplications | `O(n)` frames plus growing arithmetic state | `Theta(n log n)` bits broadly |
| 133 | Memoized Fibonacci | `O(n)` bigint additions | `O(n)` | `Theta(n)` bits |
| 134 | Recursive digit reduction | `O(d)` | `O(d)` | `O(1)` scalar |
| 135 | Recursive Unicode code-point reversal | `O(n)` | `O(n)` | `O(n)` |
| 136 | Recursive palindrome comparison | `O(n)` | `O(n)` | `O(1)` |
| 137 | Subsequences | `O(n * 2^n)` | `O(n)` | `O(n * 2^n)` |
| 138 | Subsets | `O(n * 2^n)` | `O(n)` | `O(n * 2^n)` |
| 139 | Permutations | `Theta(n * n!)` | `O(n)` | `Theta(n * n!)` |
| 140 | Combination Sum | exponential, output-sensitive | `O(T)` path/stack | proportional to returned combinations |
| 141 | Phone letter combinations | `O(n * 4^n)` worst case | `O(n)` | `O(n * 4^n)` |
| 142 | Generate Parentheses | `Theta(C_n * n)` | `O(n)` | `Theta(C_n * n)` |
| 143 | Word Search | `O(R * C * 4^L)` worst case | `O(R * C + L)` | `O(1)` boolean |
| 144 | N-Queens | `O(n! + S * n^2)` | `O(n)` | `Theta(S * n^2)` |
| 145 | Sudoku with masks and MRV | worst-case exponential, at most `O(E * 9^E)` conservatively | `O(E + 81)` plus `O(E)` frames | `O(1)` beyond mutated input |

## Common Interview Traps
- Writing a base case without proving that every recursive edge reaches it.
- Forgetting that JavaScript stacks are finite and tail-call optimization is not dependable.
- Calling exact `bigint` arithmetic constant-time or returning a `bigint` directly through JSON.
- Indexing strings by UTF-16 code unit when the contract is defined in Unicode code points.
- Confusing code points with grapheme clusters or silently normalizing text.
- Memoizing an incomplete state key or caching a search with no repeated states.
- Mutating caller-owned input before complete validation.
- Emitting the same mutable path reference instead of a fresh snapshot.
- Failing to unchoose after success, failure, early return, or exception.
- Removing duplicate choices globally when duplicates should be skipped only at one depth.
- Treating a heuristic prune as if it were a proof that a branch is impossible.
- Ignoring the time and space required to materialize exponential output.
- Claiming that MRV or bit masks guarantee a better exponential asymptotic bound.
- Returning an unsolvable in-place board with partial assignments still present.
- Exposing unbounded recursion traces or full state snapshots from an API.

## Practice Checklist
- [ ] 131. Trace descent and unwind order for recursive number generation.
- [ ] 132. Compute factorial with exact `bigint` arithmetic and a validated bound.
- [ ] 133. Explain overlapping Fibonacci states and implement memoization.
- [ ] 134. Prove termination by reducing the number of remaining digits.
- [ ] 135. Reverse a string by Unicode code points without quadratic concatenation.
- [ ] 136. Check a palindrome with inward-moving recursive indexes.
- [ ] 137. Generate subsequences in deterministic exclude-before-include order.
- [ ] 138. Emit fresh subset snapshots while reusing one mutable path.
- [ ] 139. Generate permutations with reversible used-index state.
- [ ] 140. Prune combination search only under proven nonnegative bounds.
- [ ] 141. Map phone digits to choices and account for output size.
- [ ] 142. Generate only prefixes whose parenthesis counts can still be valid.
- [ ] 143. Restore grid state exactly across every Word Search exit path.
- [ ] 144. Maintain N-Queens columns and both diagonal families.
- [ ] 145. Solve Sudoku in place with atomic validation, MRV masks, and failure restoration.

