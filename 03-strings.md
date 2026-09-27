# Strings

## Learning Objectives

By the end of this topic, you should understand:

- how strings represent ordered text and why JavaScript strings are immutable
- the difference between UTF-16 code units and Unicode code points
- how normalization rules change palindrome, anagram, and counting contracts
- how to traverse, reverse, compare, compress, and partition strings
- how frequency maps and sets support character problems
- how two pointers and sliding windows process string ranges
- how expansion around centers finds palindromes
- how to include string copying and key construction in complexity analysis
- how to produce useful execution steps for a NestJS visualization API

## Prerequisites

- contracts, invariants, and complexity reasoning from `00-dsa-fundamentals.md`
- array traversal and two-pointer basics from `01-arrays.md`
- `Map` and `Set` patterns from `02-hashing.md`
- TypeScript strings, arrays, loops, and functions

## Core Theory

### What is a string?

A **string** is an ordered sequence of text symbols. Positions begin at zero:

```text
Text:   c  o  d  e
Index:  0  1  2  3
```

Strings model names, messages, identifiers, documents, paths, and encoded data.

### Why do string algorithms exist?

Text problems ask more than direct lookup. Programs often need to:

- compare text under normalization rules
- count or group characters
- find repeated or unique regions
- validate nested syntax
- identify prefixes, rotations, or palindromes
- transform text without losing intended symbols

### Strings are immutable in TypeScript

JavaScript strings cannot be changed at one position:

```typescript
const text = "code";
// text[0] = "m" does not mutate the string.
```

Algorithms build a new string or convert text to a mutable character array and join it afterward. That output or copy uses memory proportional to text length.

### What is a character in JavaScript?

JavaScript indexes strings by UTF-16 **code unit**, not always by complete user-perceived character.

```typescript
"😀".length; // 2 code units
[..."😀"].length; // 1 Unicode code point
```

This chapter commonly uses `[...text]` so emoji represented by one code point remain together. Some visible symbols combine several code points; full grapheme-cluster handling requires `Intl.Segmenter` and an explicit product requirement.

### Normalization is part of the contract

These questions produce different answers depending on policy:

- Is comparison case-sensitive?
- Are spaces and punctuation included?
- Should accented forms be Unicode-normalized?
- Is input limited to lowercase English letters?

Never silently remove or lowercase characters unless the problem states that rule.

### Strings and arrays

```text
string --spread--> character array --algorithm--> result --join--> string
```

Spreading, slicing, joining, and concatenating all perform work. Convenience methods do not make traversal free.

### Lexicographic order

Lexicographic comparison resembles dictionary order: compare symbols from left to right until they differ. Exact ordering depends on code points or locale rules. DSA examples usually use direct equality unless ordering is explicitly required.

## Visual Mental Model

### Two pointers

```text
"r a c e c a r"
  ^           ^
 left       right
```

Compare or swap corresponding outer positions, then move inward.

### Frequency state

```text
"aabbc"

a -> 2
b -> 2
c -> 1
```

### Sliding window

```text
"a b c a b c b b"
     [-----]
      current unique window
```

Move the left boundary when the right boundary introduces a duplicate.

### Palindrome centers

```text
Odd center:   c a r a c
                  ^

Even center:  a b b a
                ^ ^
```

Every palindrome has either one central symbol or a gap between two central symbols.

## Complexity Concepts

Let `n` be the number of processed code points.

| Operation | Typical time | Space |
|---|---:|---:|
| Read known string code unit | `O(1)` | `O(1)` |
| Spread to code-point array | `O(n)` | `O(n)` |
| Join `n` characters | `O(n)` | `O(n)` result |
| Compare/traverse all characters | `O(n)` | Often `O(1)` or `O(u)` |
| `slice` a length-`k` region | `O(k)` | `O(k)` result |
| Sort `n` characters | `O(n log n)` | Implementation-dependent |
| Build character frequencies | Average `O(n)` | `O(u)` |

### Repeated concatenation warning

Building a string with `result = character + result` can repeatedly copy the growing result. Use an array and one final `join` when predictable linear construction matters.

### Alphabet size matters

If input is guaranteed to use 26 lowercase English letters, a fixed 26-count array uses `O(1)` space relative to `n`. For unrestricted Unicode keys, distinct-character space is `O(u)`, up to `O(n)`.

### Output space still counts

Returning a reversed or compressed string of length proportional to `n` requires `O(n)` output space even if pointer state is constant.

## Common Patterns

### Character traversal
Visit each code point and update a count, result, or classification.

### Inward two pointers
Compare mirrored positions for reversal and palindrome checks.

### Frequency comparison
Anagrams and repeated-character problems compare or inspect character counts.

### Canonical representation
Sorted characters or frequency signatures turn equivalent strings into matching keys.

### Sliding window
Maintain a valid contiguous substring while boundaries move only forward.

### Stack validation
Nested delimiters require last-opened, first-closed behavior.

### Expand around center
Grow palindromes from every odd and even center instead of generating every substring.

---

# Problems

## Problem 36 — Reverse a String

### 1. Problem Statement
Return a new string whose Unicode code points appear in reverse order.

### 2. Example
```text
Input: "hello"
Output: "olleh"
```

### 3. Understand the Problem
The original string cannot be mutated. This contract reverses code points, not full grapheme clusters.

#### In Simple Terms

Return the same code points in the opposite order. The original string stays unchanged.

### 4. Real-World Analogy
Turn a row of letter cards around by swapping the first with last, then moving inward.

### 5. Key Observation
After converting to a mutable array, each outer swap puts two code points in final positions.

### 6. Pattern
```text
Pattern: Inward two pointers
```
### 7. Brute Force
Prepend each code point to a growing result.
#### Pseudocode
```text
result = empty
for each character
    result = character + result
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
h -> h; e -> eh; l -> leh; l -> lleh; o -> olleh
```
#### Dry Run
Five prepends produce `"olleh"`.
#### Brute-Force Complexity
Repeatedly copying a growing result can take `O(n²)` time; the result uses `O(n)` space.

### 8. Why Brute Force Is Not Ideal
Each prepend may copy all characters already accumulated.

### 9. Optimal Approach
Spread into code points, swap outer positions inward, then join once.

### 10. Visualization
```text
[h,e,l,l,o] -> [o,e,l,l,h] -> [o,l,l,e,h]
 ^       ^        ^   ^
```

### 11. Step-by-Step Dry Run
| Left | Right | Swap | State |
|---:|---:|---|---|
| 0 | 4 | h/o | `[o,e,l,l,h]` |
| 1 | 3 | e/l | `[o,l,l,e,h]` |

### 12. TypeScript Solution
```typescript
function reverseString(text: string): string {
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
Spread preserves individual code points such as a simple emoji. Pointer movement guarantees termination; joining creates the immutable result.

### 14. Complexity Analysis
**Time: `O(n)`** for spreading, at most `n/2` swaps, and joining; sequential linear work remains linear. **Space: `O(n)`** for the character array and output.

### 15. Edge Cases
- Empty and one-code-point strings return unchanged.
- Spaces and punctuation reverse normally.
- Combining grapheme sequences may visually split under this code-point contract.

### 16. Common Mistakes
- Claiming strings are reversed in place.
- Using code-unit indexing and splitting surrogate pairs.
- Calling repeated prepending linear without considering copying.

### 17. Interview Follow-ups
1. Reverse grapheme clusters. 2. Reverse a mutable character array in place. 3. Reverse only letters.

### 18. Variations
- Reverse words. - Reverse vowels only. - Reverse a substring range.

### 19. Pattern Connections
Inward pointers also support palindrome checks and symmetric transformations.

### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/reverse`; trace swap indexes and states only below a configured text-length limit.

### 21. Mini Challenge
Use `Intl.Segmenter` to reverse user-perceived grapheme clusters without splitting combined emoji.

## Problem 37 — Check if a String Is a Palindrome
### 1. Problem Statement
Return whether a string reads identically forward and backward by code point, using case-sensitive exact comparison.
### 2. Example
```text
Input: "racecar"
Output: true
```
### 3. Understand the Problem
Spaces, punctuation, and case are significant under this contract.

#### In Simple Terms

Return `true` when the exact string looks the same from left to right and right to left.

### 4. Real-World Analogy
Fold a row of symbol cards in half and compare mirrored pairs.
### 5. Key Observation
The string is a palindrome exactly when every outer pair matches.
### 6. Pattern
```text
Pattern: Inward two-pointer comparison
```
### 7. Brute Force
Reverse the string and compare it with the original.
#### Pseudocode
```text
reversed = reverse(text)
return reversed equals text
```
#### Brute-Force TypeScript

```typescript
function isPalindromeByReversing(text: string): boolean {
  return [...text].reverse().join("") === text;
}
```

#### Visualization
```text
racecar -> racecar -> equal
```
#### Dry Run
The reversed copy matches all seven code points.
#### Brute-Force Complexity
Reversal and comparison take `O(n)` time and the reversed representation uses `O(n)` space.
### 8. Why Brute Force Is Not Ideal
The full reversed copy is unnecessary for a boolean answer.
### 9. Optimal Approach
Compare left and right code points, returning false at the first mismatch.
### 10. Visualization
```text
r a c e c a r
^           ^ match
  ^       ^   match
    ^   ^     match
```
### 11. Step-by-Step Dry Run
Pairs `(r,r)`, `(a,a)`, and `(c,c)` match; pointers meet at `e`, so return true.
### 12. TypeScript Solution
```typescript
function isPalindrome(text: string): boolean {
  const characters = [...text];
  let left = 0;
  let right = characters.length - 1;
  while (left < right) {
    if (characters[left] !== characters[right]) return false;
    left += 1;
    right -= 1;
  }
  return true;
}
```
### 13. Code Walkthrough
The array supports code-point positions. Early mismatch proves failure; crossing pointers proves every mirrored pair matched.
### 14. Complexity Analysis
**Time: `O(n)`** for conversion and at most half the comparisons. **Space: `O(n)`** for code points; code-unit-only input could use `O(1)` pointer space.
### 15. Edge Cases
- Empty and one-character strings return true. - Case differs. - Spaces count. - Grapheme clusters are not combined.
### 16. Common Mistakes
- Silently lowercasing or removing punctuation.
- Comparing only the first and last pair.
- Claiming constant space while spreading.
### 17. Interview Follow-ups
1. Ignore non-alphanumeric symbols. 2. Normalize case. 3. Allow one deletion.
### 18. Variations
- Palindrome number. - Linked-list palindrome. - Longest palindromic substring.
### 19. Pattern Connections
Mirrored comparison underlies symmetric arrays, trees, and near-palindrome problems.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/palindrome`; include normalization options explicitly and trace compared indexes.
### 21. Mini Challenge
Implement an alphanumeric, case-insensitive version without first creating a cleaned string.

## Problem 38 — Check if Two Strings Are Anagrams
### 1. Problem Statement
Return whether two case-sensitive strings contain exactly the same code points with the same frequencies.
### 2. Example
```text
Input: "listen", "silent"
Output: true
```
### 3. Understand the Problem
Order may differ; multiplicity may not. No case or whitespace normalization occurs.

#### In Simple Terms

Return `true` when both strings contain the same code points the same number of times, even if their order differs.

### 4. Real-World Analogy
Two bags of letter tiles are equivalent when every tile type has the same count.
### 5. Key Observation
Equal lengths and a zeroed frequency balance prove equality of the multisets.
### 6. Pattern
```text
Pattern: Character frequency balance
```
### 7. Brute Force
Sort both code-point arrays and compare them.
#### Pseudocode
```text
if lengths differ, false
sort characters of both strings
compare sorted arrays
```
#### Brute-Force TypeScript

```typescript
function areAnagramsBySorting(first: string, second: string): boolean {
  const left = [...first].sort();
  const right = [...second].sort();
  return left.length === right.length && left.every((character, index) => character === right[index]);
}
```

#### Visualization
```text
listen -> eilnst
silent -> eilnst
```
#### Dry Run
Both canonical sorted forms match.
#### Brute-Force Complexity
Sorting length `n` and `m` costs `O(n log n + m log m)` time and `O(n+m)` representation space.
### 8. Why Brute Force Is Not Ideal
Full ordering is unnecessary; only counts matter.
### 9. Optimal Approach
Increment counts for the first string and decrement while reading the second; missing or overused characters fail.
### 10. Visualization
```text
listen counts -> l1 i1 s1 t1 e1 n1
silent decrements every count to zero
```
### 11. Step-by-Step Dry Run
Length check passes. Each symbol in `silent` consumes one available count; none drops below zero.
### 12. TypeScript Solution
```typescript
function areAnagrams(first: string, second: string): boolean {
  const left = [...first];
  const right = [...second];
  if (left.length !== right.length) return false;
  const counts = new Map<string, number>();
  for (const character of left) counts.set(character, (counts.get(character) ?? 0) + 1);
  for (const character of right) {
    const count = counts.get(character) ?? 0;
    if (count === 0) return false;
    counts.set(character, count - 1);
  }
  return true;
}
```
### 13. Code Walkthrough
Equal code-point length plus successful consumption guarantees no unconsumed positive count remains.
### 14. Complexity Analysis
**Time: average `O(n+m)`** for traversal and map updates. **Space: `O(u+n+m)`** here because spread arrays and `u` distinct counts are stored.
### 15. Edge Cases
- Two empty strings are anagrams. - Duplicate letters must match. - Case and spaces matter. - Unicode normalization is not automatic.
### 16. Common Mistakes
- Using sets and losing multiplicity.
- Omitting length comparison.
- Assuming visually identical Unicode forms have identical code points.
### 17. Interview Follow-ups
1. Lowercase English fixed-array counts. 2. Case-insensitive input. 3. Group many anagrams.
### 18. Variations
- Anagram windows. - Delete characters to make anagrams. - Ransom note construction.
### 19. Pattern Connections
Frequency balance supports permutations, inventory reconciliation, and multiset equality.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/anagram`; define case/normalization DTO options and trace count deltas.
### 21. Mini Challenge
Normalize both strings with Unicode NFC and case folding, then document the changed contract.

## Problem 39 — Find the First Non-Repeating Character
### 1. Problem Statement
Return the earliest code point appearing exactly once, or `null` if none exists.
### 2. Example
```text
Input: "swiss"
Output: "w"
```
### 3. Understand the Problem
"First" means smallest code-point index, not smallest character value.

#### In Simple Terms

Return the earliest code point that appears exactly once in the whole string.

### 4. Real-World Analogy
After reading all labels, choose the earliest label no one else repeated.
### 5. Key Observation
Total frequencies establish uniqueness; a second ordered pass selects the first.
### 6. Pattern
```text
Pattern: Frequency map followed by stable selection
```
### 7. Brute Force
For each position, scan the whole string for its count.
#### Pseudocode
```text
for each character
    count it in all characters
    return first count of one
```
#### Brute-Force TypeScript

```typescript
function firstUniqueCharacterByCounting(text: string): string | null {
  const characters = [...text];
  for (const candidate of characters) {
    let count = 0;
    for (const character of characters) {
      if (character === candidate) count += 1;
    }
    if (count === 1) return candidate;
  }
  return null;
}
```

#### Visualization
```text
s count 3; w count 1 -> answer
```
#### Dry Run
`s` fails; `w` succeeds before later unique `i`.
#### Brute-Force Complexity
Up to `n` scans of `n` code points give `O(n²)` time and `O(n)` spread space.
### 8. Why Brute Force Is Not Ideal
The same frequencies are recomputed.
### 9. Optimal Approach
Count once, then scan in original order for frequency one.
### 10. Visualization
```text
counts s:3 w:1 i:1
order s(x), w(1) -> answer
```
### 11. Step-by-Step Dry Run
The map is built in five updates. The selection pass checks `s`, then returns `w`.
### 12. TypeScript Solution
```typescript
function firstUniqueCharacter(text: string): string | null {
  const characters = [...text];
  const counts = new Map<string, number>();
  for (const character of characters) counts.set(character, (counts.get(character) ?? 0) + 1);
  for (const character of characters) if (counts.get(character) === 1) return character;
  return null;
}
```
### 13. Code Walkthrough
The map answers global frequency; iterating the original sequence preserves ordering.
### 14. Complexity Analysis
**Time: average `O(n)`** across two passes. **Space: `O(n+u)`**, simplified to `O(n)`, for code points and counts.
### 15. Edge Cases
- Empty/all-repeated returns null. - One symbol returns itself. - Case-sensitive keys differ.
### 16. Common Mistakes
- Returning before future repeats are known.
- Iterating sorted keys instead of input order.
- Using only a set.
### 17. Interview Follow-ups
1. Return index. 2. Streaming first unique character. 3. Return all unique characters.
### 18. Variations
- First unique word. - First frequency-`k` symbol. - Queue-based stream processing.
### 19. Pattern Connections
Frequency qualification plus stable selection matches the hashing chapter's array version.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/first-unique`; return count and selection trace phases.
### 21. Mini Challenge
Return the code-point index and explain why it may differ from the UTF-16 code-unit index.

## Problem 40 — Find the First Repeating Character
### 1. Problem Statement
Return the repeating character whose first occurrence is earliest, or `null` if all are unique.
### 2. Example
```text
Input: "abccba"
Output: "a"
```
### 3. Understand the Problem
Although `c` is the first duplicate encountered, `a` is the answer because its first occurrence is earliest.

#### In Simple Terms

Among code points used more than once, return the one whose first copy appears earliest.

### 4. Real-World Analogy
Among labels used more than once, choose the one originally introduced first.
### 5. Key Observation
Count globally, then select the first ordered character with frequency above one.
### 6. Pattern
```text
Pattern: Frequency map followed by stable selection
```
### 7. Brute Force
For each position, search later positions for the same character.
#### Pseudocode
```text
for i from left
    for j after i
        if equal, return character[i]
```
#### Brute-Force TypeScript

```typescript
function firstRepeatingCharacterBySuffixSearch(text: string): string | null {
  const characters = [...text];
  for (let first = 0; first < characters.length - 1; first += 1) {
    for (let second = first + 1; second < characters.length; second += 1) {
      if (characters[first] === characters[second]) return characters[first];
    }
  }
  return null;
}
```

#### Visualization
```text
a searches right -> finds final a -> answer
```
#### Dry Run
The first outer position already has a later match.
#### Brute-Force Complexity
No-repeat input checks `O(n²)` pairs; spread representation costs `O(n)` space.
### 8. Why Brute Force Is Not Ideal
Later suffixes are repeatedly searched.
### 9. Optimal Approach
Build frequencies, then return the first character whose count exceeds one.
### 10. Visualization
```text
counts a:2 b:2 c:2
ordered scan starts at a -> answer
```
### 11. Step-by-Step Dry Run
The count pass records two of each. Selection stops at code-point index zero.
### 12. TypeScript Solution
```typescript
function firstRepeatingCharacter(text: string): string | null {
  const characters = [...text];
  const counts = new Map<string, number>();
  for (const character of characters) counts.set(character, (counts.get(character) ?? 0) + 1);
  for (const character of characters) if ((counts.get(character) ?? 0) > 1) return character;
  return null;
}
```
### 13. Code Walkthrough
The two passes separate repeat qualification from earliest-first-occurrence selection.
### 14. Complexity Analysis
**Time: average `O(n)`**; **space: `O(n)`** including code points and up to `n` distinct map keys.
### 15. Edge Cases
- Empty or all-unique returns null. - All equal returns that symbol. - Case remains significant.
### 16. Common Mistakes
- Returning the first duplicate encountered (`c` in the example).
- Confusing value with index.
- Using a set without preserving the required selection rule.
### 17. Interview Follow-ups
1. Return earliest second occurrence. 2. Return both indexes. 3. Stream duplicate detection.
### 18. Variations
- First repeating word. - First character repeated `k` times. - Nearby repeats.
### 19. Pattern Connections
The distinction between earliest first occurrence and earliest repeat event matters in stable duplicate problems.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/first-repeating`; label selection semantics in response metadata.
### 21. Mini Challenge
Return `{ character, firstIndex, secondIndex }` for the selected repeating character.

## Problem 41 — Count Vowels and Consonants
### 1. Problem Statement
Count vowels and consonants among English letters, case-insensitively; ignore all other code points.
### 2. Example
```text
Input: "Hello, DSA!"
Output: { vowels: 3, consonants: 5 }
```
### 3. Understand the Problem
Vowels are `a,e,i,o,u`. `y` is a consonant; spaces, punctuation, digits, and non-English letters are ignored.

#### In Simple Terms

Count English vowels and English consonants separately, ignoring anything that is not an English letter.

### 4. Real-World Analogy
Sort letter tiles into vowel and consonant containers while discarding non-letter tokens.
### 5. Key Observation
Each symbol can be classified independently in one traversal.
### 6. Pattern
```text
Pattern: Character classification and accumulation
```
### 7. Brute Force
Run separate full passes to count each vowel and then derive consonants with another letter pass.
#### Pseudocode
```text
count a, then e, then i, then o, then u
scan again for all English letters
```
#### Brute-Force TypeScript

```typescript
function countVowelsAndConsonantsWithRepeatedPasses(
  text: string,
): { vowels: number; consonants: number } {
  const characters = [...text].map((character) => character.toLowerCase());
  let vowels = 0;
  for (const vowel of ["a", "e", "i", "o", "u"]) {
    for (const character of characters) {
      if (character === vowel) vowels += 1;
    }
  }
  let letters = 0;
  for (const character of characters) {
    if (character >= "a" && character <= "z") letters += 1;
  }
  return { vowels, consonants: letters - vowels };
}
```

#### Visualization
```text
many passes over the same text for separate categories
```
#### Dry Run
Six scans eventually produce vowels `3` and consonants `5`.
#### Brute-Force Complexity
A constant six passes are `O(n)` time, but repeat work; converted/lowercased output may use `O(n)` space.
### 8. Why Brute Force Is Not Ideal
One decision per character can update the correct counter immediately.
### 9. Optimal Approach
Lowercase each code point, check ASCII letter range, then test vowel-set membership.
### 10. Visualization
```text
H -> h consonant; e vowel; l consonant; l consonant; o vowel
, ignored; D consonant; S consonant; A vowel
```
### 11. Step-by-Step Dry Run
The three vowels are `e,o,A`; consonants are `H,l,l,D,S`.
### 12. TypeScript Solution
```typescript
function countVowelsAndConsonants(text: string): { vowels: number; consonants: number } {
  const vowelSet = new Set(["a", "e", "i", "o", "u"]);
  let vowels = 0;
  let consonants = 0;
  for (const symbol of text) {
    const character = symbol.toLowerCase();
    if (character < "a" || character > "z") continue;
    if (vowelSet.has(character)) vowels += 1;
    else consonants += 1;
  }
  return { vowels, consonants };
}
```
### 13. Code Walkthrough
The range check enforces the English-only contract; every accepted letter increments exactly one counter.
### 14. Complexity Analysis
**Time: `O(n)`** for one traversal. **Auxiliary space: `O(1)`** because the vowel set has fixed size and no input-sized copy is stored.
### 15. Edge Cases
- Empty/non-letter-only returns both zero. - Uppercase works. - Accented letters are ignored by contract.
### 16. Common Mistakes
- Counting spaces as consonants.
- Forgetting uppercase.
- Treating `y` inconsistently.
- Claiming general Unicode letter support.
### 17. Interview Follow-ups
1. Support Unicode letter categories. 2. Return per-vowel counts. 3. Treat `y` contextually.
### 18. Variations
- Count digits/whitespace. - Character histogram. - Word-level classification.
### 19. Pattern Connections
Classification traversals support parsers, validators, and text analytics.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/count-letters`; return classification rules and trace `{ symbol, category, totals }`.
### 21. Mini Challenge
Use Unicode property escapes to count all Unicode letters while keeping vowel policy explicit.

## Problem 42 — Remove Duplicate Characters
### 1. Problem Statement
Return a string containing only the first occurrence of each code point, preserving encounter order.
### 2. Example
```text
Input: "programming"
Output: "progamin"
```
### 3. Understand the Problem
Comparison is case-sensitive; the input is unchanged.

#### In Simple Terms

Keep the first copy of each code point and remove all later copies without changing the kept order.

### 4. Real-World Analogy
Keep the first copy of each letter card and discard later copies.
### 5. Key Observation
A set answers whether the current character was already accepted.
### 6. Pattern
```text
Pattern: Seen set plus stable output
```
### 7. Brute Force
For each character, linearly search the result built so far.
#### Pseudocode
```text
result = []
for character
    append if result does not include it
```
#### Brute-Force TypeScript

```typescript
function removeDuplicateCharactersBySearching(text: string): string {
  const accepted: string[] = [];
  for (const character of text) {
    if (!accepted.includes(character)) accepted.push(character);
  }
  return accepted.join("");
}
```

#### Visualization
```text
p -> p; r -> pr; o -> pro; g -> prog; r -> skip
```
#### Dry Run
Continuing through all letters produces `progamin`.
#### Brute-Force Complexity
`includes` may scan an `O(n)` result for `n` characters: `O(n²)` time and `O(n)` output space.
### 8. Why Brute Force Is Not Ideal
Accepted characters are rescanned repeatedly.
### 9. Optimal Approach
Use a seen set and append unseen code points to an array.
### 10. Visualization
```text
seen {p,r,o,g}; second r -> skip; a -> append...
```
### 11. Step-by-Step Dry Run
First occurrences append in order: `p,r,o,g,a,m,i,n`; all repeated `r`, `m`, and `g` are skipped.
### 12. TypeScript Solution
```typescript
function removeDuplicateCharacters(text: string): string {
  const seen = new Set<string>();
  const result: string[] = [];
  for (const character of text) {
    if (seen.has(character)) continue;
    seen.add(character);
    result.push(character);
  }
  return result.join("");
}
```
### 13. Code Walkthrough
Iteration visits code points. The set controls membership while the array preserves stable output and enables one final join.
### 14. Complexity Analysis
**Time: average `O(n)`** for set operations plus join. **Space: `O(u)` auxiliary and `O(u)` output**, worst `O(n)`.
### 15. Edge Cases
- Empty returns empty. - All equal returns one symbol. - Case variants remain distinct.
### 16. Common Mistakes
- Sorting and losing encounter order.
- Using repeated string concatenation without considering copies.
- Removing only adjacent duplicates.
### 17. Interview Follow-ups
1. Keep last occurrences. 2. Ignore case while preserving original symbol. 3. Remove adjacent duplicate runs only.
### 18. Variations
- Deduplicate words. - Stable array deduplication. - Frequency-one characters only.
### 19. Pattern Connections
Stable set filtering is shared with unsorted array deduplication.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/remove-duplicates`; trace accepted/skipped status without serializing the full set.
### 21. Mini Challenge
Keep the last occurrence of each character while preserving the order of those retained positions.

## Problem 43 — Reverse Words in a String
### 1. Problem Statement
Reverse word order, treating one or more whitespace characters as separators and joining output with single spaces.
### 2. Example
```text
Input: "  learn   DSA now "
Output: "now DSA learn"
```
### 3. Understand the Problem
Characters inside each word retain order; leading, trailing, and repeated whitespace are normalized.

#### In Simple Terms

Put the last word first and the first word last. Keep letters inside each word unchanged and use one space between output words.

### 4. Real-World Analogy
Reverse the order of labeled boxes without rearranging letters printed on each box.
### 5. Key Observation
Tokenize words, reverse the token sequence, and join with the contract's separator.
### 6. Pattern
```text
Pattern: Tokenize, transform, join
```
### 7. Brute Force
Scan backward, repeatedly prepend discovered words to a result string.
#### Pseudocode
```text
find words from right to left
concatenate each plus separator
```
#### Brute-Force TypeScript

```typescript
function reverseWordsByBackwardScanning(text: string): string {
  const characters = [...text];
  let result = "";
  let end = characters.length - 1;
  while (end >= 0) {
    while (end >= 0 && /\s/u.test(characters[end])) end -= 1;
    if (end < 0) break;
    let start = end;
    while (start >= 0 && !/\s/u.test(characters[start])) start -= 1;
    const word = characters.slice(start + 1, end + 1).join("");
    result += result === "" ? word : ` ${word}`;
    end = start;
  }
  return result;
}
```

#### Visualization
```text
discover now, DSA, learn -> append in that order
```
#### Dry Run
Backward scanning identifies three words and constructs the expected output.
#### Brute-Force Complexity
Careless repeated concatenation can copy growing text for `O(n²)` time; output is `O(n)`.
### 8. Why Brute Force Is Not Ideal
Manual boundaries and repeated copies add complexity under the normalization contract.
### 9. Optimal Approach
Trim, split on one-or-more whitespace, reverse tokens, and join once.
### 10. Visualization
```text
"learn DSA now" -> [learn,DSA,now] -> [now,DSA,learn] -> "now DSA learn"
```
### 11. Step-by-Step Dry Run
Trimming removes boundaries; regex splitting creates three tokens; reversing swaps token positions; join inserts two spaces.
### 12. TypeScript Solution
```typescript
function reverseWords(text: string): string {
  const trimmed = text.trim();
  if (trimmed === "") return "";
  return trimmed.split(/\s+/u).reverse().join(" ");
}
```
### 13. Code Walkthrough
The empty guard avoids `split` producing an empty token. The regex implements the whitespace-normalization contract.
### 14. Complexity Analysis
**Time: `O(n)`** for trim, tokenization, reversal, and join as sequential linear work. **Space: `O(n)`** for tokens and output.
### 15. Edge Cases
- Empty/whitespace-only returns empty. - One word returns itself. - Tabs/newlines become single spaces.
### 16. Common Mistakes
- Reversing characters rather than words.
- Leaving repeated spaces accidentally.
- Claiming in-place mutation of an immutable string.
### 17. Interview Follow-ups
1. Preserve exact whitespace. 2. Reverse words in a mutable buffer. 3. Reverse characters in each word.
### 18. Variations
- Sentence token reversal. - Path segment reversal. - Word-order rotation.
### 19. Pattern Connections
Tokenize-transform-join is common in parsers and text normalization pipelines.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/reverse-words`; return tokenization and reorder phases.
### 21. Mini Challenge
Preserve every original whitespace run while reversing only word positions.

## Problem 44 — Find the Longest Word in a String
### 1. Problem Statement
Return the first longest whitespace-delimited word, or `null` when no word exists.
### 2. Example
```text
Input: "DSA builds problem solving"
Output: "problem"
```
### 3. Understand the Problem
Punctuation attached to a token counts as part of it. Ties keep the earliest word.

#### In Simple Terms

Split the text into words and return the first word with the greatest code-point length.

### 4. Real-World Analogy
Scan labeled cards and retain the first card longer than every earlier card.
### 5. Key Observation
Only the best word seen so far is needed; strict `>` preserves the first tie winner.
### 6. Pattern
```text
Pattern: Token traversal with running best
```
### 7. Brute Force
Sort all words by descending length and return the first.
#### Pseudocode
```text
tokenize words
stable-sort by length descending
return first
```
#### Brute-Force TypeScript

```typescript
function longestWordBySorting(text: string): string | null {
  const trimmed = text.trim();
  if (trimmed === "") return null;
  const words = trimmed.split(/\s+/u).map((word, index) => ({
    word,
    index,
    length: [...word].length,
  }));
  words.sort((first, second) => second.length - first.length || first.index - second.index);
  return words[0].word;
}
```

#### Visualization
```text
DSA(3), builds(6), problem(7), solving(7) -> problem first among max
```
#### Dry Run
Sorting places both seven-character words first; stable order keeps `problem` first.
#### Brute-Force Complexity
For `w` words, sorting costs `O(w log w)` comparisons plus `O(n)` tokenization and storage.
### 8. Why Brute Force Is Not Ideal
Ordering every word is unnecessary to select one maximum.
### 9. Optimal Approach
Tokenize once and replace `longest` only when a strictly longer word appears.
### 10. Visualization
```text
DSA -> DSA; builds -> builds; problem -> problem; solving ties -> keep problem
```
### 11. Step-by-Step Dry Run
Lengths `3,6,7,7` update the best at the first three tokens only.
### 12. TypeScript Solution
```typescript
function longestWord(text: string): string | null {
  const trimmed = text.trim();
  if (trimmed === "") return null;
  let longest = "";
  let longestLength = 0;
  for (const word of trimmed.split(/\s+/u)) {
    const length = [...word].length;
    if (length > longestLength) {
      longest = word;
      longestLength = length;
    }
  }
  return longest;
}
```
### 13. Code Walkthrough
The cached code-point length avoids rescanning the current best word. The strict comparison defines tie behavior.
### 14. Complexity Analysis
**Time: `O(n)`** because tokenization and each token's one code-point count process total input length linearly. **Space: `O(n)`** for tokens.
### 15. Edge Cases
- Empty/whitespace-only returns null. - First tie wins. - Punctuation is included. - Code-point length differs from grapheme length.
### 16. Common Mistakes
- Replacing on `>=` and returning the last tie.
- Sorting unnecessarily.
- Leaving tokenization rules undefined.
### 17. Interview Follow-ups
1. Ignore punctuation. 2. Return all longest words. 3. Process a stream without token array.
### 18. Variations
- Shortest word. - Longest unique word. - Longest line in text.
### 19. Pattern Connections
Running-best traversal matches array extrema and longest-run state.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/longest-word`; trace `{ word, length, best }` and document tokenizer policy.
### 21. Mini Challenge
Rewrite tokenization as a single pass so auxiliary storage excludes an array of all words.

## Problem 45 — Check if One String Is a Rotation of Another
### 1. Problem Statement
Return whether `candidate` can be produced by moving a prefix of `source` to its end.
### 2. Example
```text
Input: "waterbottle", "erbottlewat"
Output: true
```
### 3. Understand the Problem
Lengths must match. The comparison is exact and case-sensitive; empty strings are rotations of each other.

#### In Simple Terms

Return `true` when cutting some prefix from `source` and moving it to the end can produce `candidate`.

### 4. Real-World Analogy
Read the same labels around a circular bracelet from a different starting bead.
### 5. Key Observation
Every rotation of `source` appears as a length-`n` substring of `source + source`.
### 6. Pattern
```text
Pattern: Transform rotation into substring search
```
### 7. Brute Force
Generate each of `n` rotations and compare it with the candidate.
#### Pseudocode
```text
for split from 0 to n-1
    rotation = suffix + prefix
    compare with candidate
```
#### Brute-Force TypeScript

```typescript
function isRotationByGeneratingRotations(source: string, candidate: string): boolean {
  const sourceCharacters = [...source];
  const candidateCharacters = [...candidate];
  if (sourceCharacters.length !== candidateCharacters.length) return false;
  if (sourceCharacters.length === 0) return true;
  for (let split = 0; split < sourceCharacters.length; split += 1) {
    const rotation = [...sourceCharacters.slice(split), ...sourceCharacters.slice(0, split)];
    if (rotation.every((character, index) => character === candidateCharacters[index])) return true;
  }
  return false;
}
```

#### Visualization
```text
waterbottle -> aterbottlew -> ... -> erbottlewat
```
#### Dry Run
The rotation beginning at `e` matches after several generated copies.
#### Brute-Force Complexity
Creating/comparing `n` length-`n` rotations takes `O(n²)` time and `O(n)` temporary space.
### 8. Why Brute Force Is Not Ideal
Most rotation characters are copied repeatedly.
### 9. Optimal Approach
After equal-length validation, search `candidate` inside doubled `source` with KMP, which preprocesses the pattern to avoid restarting comparisons.
### 10. Visualization
```text
source+source = waterbottlewaterbottle
                    [erbottlewat] found
```
### 11. Step-by-Step Dry Run
KMP builds fallback lengths for `erbottlewat`, scans the doubled source left to right, and reaches a full pattern match without moving the text index backward.
### 12. TypeScript Solution
```typescript
function containsKmp(text: string[], pattern: string[]): boolean {
  if (pattern.length === 0) return true;
  const prefix = Array(pattern.length).fill(0);
  for (let i = 1, length = 0; i < pattern.length;) {
    if (pattern[i] === pattern[length]) prefix[i++] = ++length;
    else if (length > 0) length = prefix[length - 1];
    else prefix[i++] = 0;
  }
  for (let i = 0, j = 0; i < text.length;) {
    if (text[i] === pattern[j]) { i += 1; j += 1; if (j === pattern.length) return true; }
    else if (j > 0) j = prefix[j - 1];
    else i += 1;
  }
  return false;
}

function isRotation(source: string, candidate: string): boolean {
  const sourceChars = [...source];
  const candidateChars = [...candidate];
  if (sourceChars.length !== candidateChars.length) return false;
  return containsKmp([...sourceChars, ...sourceChars], candidateChars);
}
```
### 13. Code Walkthrough
Equal lengths prevent partial matches. The prefix table records safe pattern fallback positions, keeping both preprocessing and search linear.
### 14. Complexity Analysis
**Time: `O(n)`** for conversion, doubling, KMP preprocessing, and scanning. **Space: `O(n)`** for arrays, doubled text, and prefix table.
### 15. Edge Cases
- Two empty strings return true. - Different lengths return false. - Identical strings are rotations. - Case matters.
### 16. Common Mistakes
- Omitting equal-length check.
- Assuming built-in substring search has a specified linear guarantee.
- Generating every rotation while claiming linear time.
### 17. Interview Follow-ups
1. Return rotation offset. 2. Use built-in search and discuss guarantees. 3. Handle grapheme clusters.
### 18. Variations
- Rotated arrays. - Circular pattern search. - Minimal string rotation.
### 19. Pattern Connections
Transform-and-search connects circular structures with substring algorithms such as KMP.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/is-rotation`; trace KMP prefix construction and match/fallback events with limits.
### 21. Mini Challenge
Return the smallest rotation offset that produces `candidate`, including repeated-pattern strings.

## Problem 46 — Find the Longest Common Prefix
### 1. Problem Statement
Return the longest prefix shared by every string in an array; return `""` when none exists.
### 2. Example
```text
Input: ["flower","flow","flight"]
Output: "fl"
```
### 3. Understand the Problem
A prefix begins at index zero. Comparison is exact and case-sensitive.

#### In Simple Terms

Return the longest starting piece that every string shares. If their first code points already differ, return an empty string.

### 4. Real-World Analogy
Find the longest opening sequence printed identically on every label.
### 5. Key Observation
The shortest string limits prefix length. At each position, one mismatch ends all longer possibilities.
### 6. Pattern
```text
Pattern: Vertical character comparison
```
### 7. Brute Force
Generate prefixes of the first string from longest to shortest and test every string with `startsWith`.
#### Pseudocode
```text
for prefix lengths from first.length down to 0
    return first prefix accepted by every string
```
#### Brute-Force TypeScript

```typescript
function longestCommonPrefixByTryingCandidates(words: string[]): string {
  if (words.length === 0) return "";
  const rows = words.map((word) => [...word]);
  const first = rows[0];
  for (let length = first.length; length >= 0; length -= 1) {
    const prefix = first.slice(0, length);
    const shared = rows.every((row) => prefix.every((character, index) => row[index] === character));
    if (shared) return prefix.join("");
  }
  return "";
}
```

#### Visualization
```text
flower -> flower? no -> flowe? no -> ... -> fl? yes for all
```
#### Dry Run
Several long candidates fail before `fl` succeeds.
#### Brute-Force Complexity
Repeated construction and comparison of overlapping prefixes can take `O(nm²)` time for `n` strings of length up to `m`; candidate space is `O(m)`.
### 8. Why Brute Force Is Not Ideal
The same opening characters are compared for many candidate lengths.
### 9. Optimal Approach
Choose a shortest string, then compare each of its code-point positions across every other string until a mismatch.
### 10. Visualization
```text
flower
flow
flight
^^ shared; third column differs
```
### 11. Step-by-Step Dry Run
Column `0` is `f` everywhere; column `1` is `l`; column `2` contains `o,o,i`, so return `fl`.
### 12. TypeScript Solution
```typescript
function longestCommonPrefix(words: string[]): string {
  if (words.length === 0) return "";
  const characterRows = words.map((word) => [...word]);
  const shortest = characterRows.reduce((best, row) => row.length < best.length ? row : best);
  let length = 0;
  while (length < shortest.length) {
    const expected = shortest[length];
    if (characterRows.some((row) => row[length] !== expected)) break;
    length += 1;
  }
  return shortest.slice(0, length).join("");
}
```
### 13. Code Walkthrough
The shortest row provides safe bounds. The first failed column proves no longer common prefix can exist.
### 14. Complexity Analysis
Let `T` be total input code points. **Time: `O(T)`** for conversion and at most the shared-prefix columns across all words. **Space: `O(T)`** for code-point rows.
### 15. Edge Cases
- Empty array or any empty word returns empty. - One word returns itself. - Case matters.
### 16. Common Mistakes
- Searching for a common substring instead of prefix.
- Reading beyond the shortest word.
- Continuing after a mismatch.
### 17. Interview Follow-ups
1. Reduce auxiliary space for ASCII. 2. Use divide and conquer. 3. Repeated prefix queries with a trie.
### 18. Variations
- Longest common suffix. - Trie prefix search. - Common directory path.
### 19. Pattern Connections
Vertical comparison and tries both exploit shared beginnings.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/longest-common-prefix`; trace columns and mismatch location.
### 21. Mini Challenge
Return both the prefix and the first word/index that breaks the next character.

## Problem 47 — String Compression
### 1. Problem Statement
Run-length encode consecutive equal code points as the character followed by its count.
### 2. Example
```text
Input: "aabcccccaaa"
Output: "a2b1c5a3"
```
### 3. Understand the Problem
Only adjacent runs combine. Counts are always included, even `1`; this function returns a new string.

#### In Simple Terms

Replace each consecutive run with its code point and run length: for example, `aaa` becomes `a3`.

### 4. Real-World Analogy
Inventory consecutive identical boxes by writing one label and the run quantity.
### 5. Key Observation
A run ends when the next character differs or the input ends; emit once per run.
### 6. Pattern
```text
Pattern: Run-length traversal
```
### 7. Brute Force
For each unprocessed position, scan forward for the run and concatenate directly to the result string.
#### Pseudocode
```text
find run length
result = result + character + count
move past run
```
#### Brute-Force TypeScript

```typescript
function compressStringByConcatenating(text: string): string {
  const characters = [...text];
  let compressed = "";
  for (let start = 0; start < characters.length;) {
    let end = start + 1;
    while (end < characters.length && characters[end] === characters[start]) end += 1;
    compressed += characters[start] + String(end - start);
    start = end;
  }
  return compressed;
}
```

#### Visualization
```text
aa -> a2; b -> b1; ccccc -> c5; aaa -> a3
```
#### Dry Run
Four runs concatenate into `a2b1c5a3`.
#### Brute-Force Complexity
Run discovery is `O(n)`, but repeated immutable concatenation can copy growing output for `O(n²)` time; output is `O(n)`.
### 8. Why Brute Force Is Not Ideal
The traversal is good, but direct repeated concatenation can be costly.
### 9. Optimal Approach
Accumulate character/count pieces in an array and join once.
### 10. Visualization
```text
parts ["a","2"] -> ["a","2","b","1"] -> ... -> join
```
### 11. Step-by-Step Dry Run
Run boundaries occur before `b`, before `c`, before final `a`, and at input end; each appends two parts.
### 12. TypeScript Solution
```typescript
function compressString(text: string): string {
  const characters = [...text];
  const parts: string[] = [];
  for (let start = 0; start < characters.length;) {
    let end = start + 1;
    while (end < characters.length && characters[end] === characters[start]) end += 1;
    parts.push(characters[start], String(end - start));
    start = end;
  }
  return parts.join("");
}
```
### 13. Code Walkthrough
`[start,end)` describes one run. Setting `start=end` guarantees every code point belongs to exactly one emitted run.
### 14. Complexity Analysis
**Time: `O(n)`** because `end` moves forward across each input once and output digits total `O(n)`. **Space: `O(n)`** for characters, parts, and result.
### 15. Edge Cases
- Empty returns empty. - One symbol becomes symbol plus `1`. - Non-adjacent equal symbols form separate runs.
### 16. Common Mistakes
- Forgetting the final run.
- Grouping all equal characters globally.
- Omitting count one without stating that contract.
- Claiming in-place mutation.
### 17. Interview Follow-ups
1. Return original if compression is not shorter. 2. Compress a mutable array in place. 3. Decode safely.
### 18. Variations
- Count-and-say. - Binary run encoding. - Repeated-word compression.
### 19. Pattern Connections
Run boundaries appear in deduplication, chunking, and change-point detection.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/compress`; trace `{ start, end, character, count }` and limit expanded input/output sizes.
### 21. Mini Challenge
Implement a decoder with validation for malformed or excessively large counts.

## Problem 48 — Valid Parentheses
### 1. Problem Statement
Given only `()[]{}`, return whether every opener is closed by the correct type in the correct order.
### 2. Example
```text
Input: "{[()]}"
Output: true
```
### 3. Understand the Problem
Matching counts are insufficient; nesting order matters. Empty input is valid.

#### In Simple Terms

Return `true` only when every bracket closes the most recently opened matching bracket and nothing remains unclosed.

### 4. Real-World Analogy
Nested boxes must close in reverse order: the most recently opened box closes first.
### 5. Key Observation
The latest unmatched opener is the only one the next closer may legally match.
### 6. Pattern
```text
Pattern: Stack for nested structure
```
### 7. Brute Force
Repeatedly remove adjacent matching pairs until no change remains.
#### Pseudocode
```text
while text contains (), [], or {}
    remove matching adjacent pairs
return text is empty
```
#### Brute-Force TypeScript

```typescript
function validParenthesesByRemovingPairs(text: string): boolean {
  let current = text;
  while (true) {
    const next = current.replaceAll("()", "").replaceAll("[]", "").replaceAll("{}", "");
    if (next === current) return current === "";
    current = next;
  }
}
```

#### Visualization
```text
{[()]} -> {[]} -> {} -> empty
```
#### Dry Run
Three replacement rounds remove the innermost pair outward.
#### Brute-Force Complexity
Up to `O(n)` rounds may each scan/copy `O(n)` text: `O(n²)` time and `O(n)` space.
### 8. Why Brute Force Is Not Ideal
It repeatedly rebuilds the string instead of remembering unmatched openers.
### 9. Optimal Approach
Push expected closing symbols for openers. A closer must equal the stack's popped top; finish with an empty stack.
### 10. Visualization
```text
{ push }; [ push ]; ( push )
) pop match; ] pop match; } pop match
```
### 11. Step-by-Step Dry Run
| Symbol | Stack after |
|---|---|
| `{` | `}` |
| `[` | `}, ]` |
| `(` | `}, ], )` |
| `)` | `}, ]` |
| `]` | `}` |
| `}` | empty |
### 12. TypeScript Solution
```typescript
function validParentheses(text: string): boolean {
  const expected: string[] = [];
  const closingByOpening: Record<string, string> = { "(": ")", "[": "]", "{": "}" };
  for (const symbol of text) {
    if (symbol in closingByOpening) expected.push(closingByOpening[symbol]);
    else if (expected.pop() !== symbol) return false;
  }
  return expected.length === 0;
}
```
### 13. Code Walkthrough
Storing expected closers simplifies comparison. A premature, wrong-type, or out-of-order closer fails immediately; leftover expectations fail at the end.
### 14. Complexity Analysis
**Time: `O(n)`** because each symbol is pushed or popped once. **Space: `O(n)`** for all-open input.
### 15. Edge Cases
- Empty returns true. - A closer first returns false. - Unclosed openers return false. - Input contract excludes other symbols.
### 16. Common Mistakes
- Checking only equal counts.
- Using a queue instead of stack.
- Forgetting leftover openers.
- Silently accepting unspecified characters.
### 17. Interview Follow-ups
1. Ignore non-bracket text. 2. Return mismatch index. 3. Support quotes/comments in source code.
### 18. Variations
- Minimum additions. - Longest valid parentheses. - Generate valid parentheses.
### 19. Pattern Connections
Stacks validate HTML-like nesting, evaluate expressions, and support undo.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/valid-parentheses`; trace push/pop actions and stop at first error.
### 21. Mini Challenge
Return `{ valid, errorIndex, expected, actual }` for invalid input.

## Problem 49 — Longest Substring Without Repeating Characters
### 1. Problem Statement
Return the maximum code-point length of a contiguous substring containing no repeated character.
### 2. Example
```text
Input: "abcabcbb"
Output: 3
```
### 3. Understand the Problem
Substring means contiguous. Return a length, and treat code points case-sensitively.

#### In Simple Terms

Find the longest continuous piece in which no code point appears twice, and return that piece's length.

### 4. Real-World Analogy
Keep a viewing window over unique labels; when a label repeats, move the left edge past its previous copy.
### 5. Key Observation
The last index of a repeated character tells exactly how far left must jump, never backward.
### 6. Pattern
```text
Pattern: Variable sliding window with last-seen indexes
```
### 7. Brute Force
Start at every position and extend until a repeated character appears.
#### Pseudocode
```text
for each start
    seen = empty
    extend end until duplicate
```
#### Brute-Force TypeScript

```typescript
function longestUniqueSubstringByEveryStart(text: string): number {
  const characters = [...text];
  let best = 0;
  for (let start = 0; start < characters.length; start += 1) {
    const seen = new Set<string>();
    for (let end = start; end < characters.length; end += 1) {
      if (seen.has(characters[end])) break;
      seen.add(characters[end]);
      best = Math.max(best, end - start + 1);
    }
  }
  return best;
}
```

#### Visualization
```text
start a -> abc then next a stops; start b -> bca then next b stops
```
#### Dry Run
Several starts find length three while rebuilding their seen sets.
#### Brute-Force Complexity
Up to `O(n)` extension per start gives `O(n²)` time and `O(u)` temporary space.
### 8. Why Brute Force Is Not Ideal
Overlapping windows rescan the same characters.
### 9. Optimal Approach
Move right once. If its character was seen inside the current window, jump left to one beyond that index; update last seen and best.
### 10. Visualization
```text
abc -> best 3
next a seen at 0 -> left jumps 1 -> bca
next b seen at 1 -> left jumps 2 -> cab
```
### 11. Step-by-Step Dry Run
| Right | Char | Left after | Window length | Best |
|---:|---|---:|---:|---:|
| 0 | a | 0 | 1 | 1 |
| 1 | b | 0 | 2 | 2 |
| 2 | c | 0 | 3 | 3 |
| 3 | a | 1 | 3 | 3 |
| 4 | b | 2 | 3 | 3 |
### 12. TypeScript Solution
```typescript
function longestUniqueSubstringLength(text: string): number {
  const characters = [...text];
  const lastIndex = new Map<string, number>();
  let left = 0;
  let best = 0;
  for (let right = 0; right < characters.length; right += 1) {
    const previous = lastIndex.get(characters[right]);
    if (previous !== undefined && previous >= left) left = previous + 1;
    lastIndex.set(characters[right], right);
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```
### 13. Code Walkthrough
The `previous >= left` guard prevents moving left backward because of an occurrence outside the current window.
### 14. Complexity Analysis
**Time: average `O(n)`** because right advances once and left only moves forward. **Space: `O(n)`** for code points and last indexes.
### 15. Edge Cases
- Empty returns zero. - All equal returns one. - All unique returns length. - Case variants differ.
### 16. Common Mistakes
- Moving left backward.
- Using a subsequence instead of substring.
- Checking mapped index by truthiness and losing index zero.
### 17. Interview Follow-ups
1. Return the substring. 2. At most `k` distinct characters. 3. Grapheme-cluster version.
### 18. Variations
- Minimum window substring. - Longest replacement window. - Unique circular substring.
### 19. Pattern Connections
Last-seen jumps are a compact form of variable-window shrinking.
### 20. NestJS Project Connection
Use `POST /api/dsa/strings/longest-unique-substring`; trace boundaries, repeat index, and best length.
### 21. Mini Challenge
Return the earliest longest substring rather than only its length.

## Problem 50 — Longest Palindromic Substring
### 1. Problem Statement
Return the longest contiguous palindromic substring; on ties, return the earliest one.
### 2. Example
```text
Input: "babad"
Output: "bab"
```
### 3. Understand the Problem
Comparison is exact by code point. Both odd- and even-length palindromes must be considered.

#### In Simple Terms

Return the longest continuous piece that reads the same in both directions. If several tie, return the earliest one.

### 4. Real-World Analogy
Place calipers at every symbol and every gap, expanding while mirrored symbols match.
### 5. Key Observation
Every palindrome has an odd center or an even center, so expanding from all `2n-1` centers covers every possibility.
### 6. Pattern
```text
Pattern: Expand around center
```
### 7. Brute Force
Generate every substring and test each with two-pointer palindrome comparison.
#### Pseudocode
```text
for every start/end pair
    test substring for palindrome
    update longest
```
#### Brute-Force TypeScript

```typescript
function longestPalindromeByTestingAllSubstrings(text: string): string {
  const characters = [...text];
  let bestStart = 0;
  let bestLength = 0;
  for (let start = 0; start < characters.length; start += 1) {
    for (let end = start; end < characters.length; end += 1) {
      let left = start;
      let right = end;
      while (left < right && characters[left] === characters[right]) {
        left += 1;
        right -= 1;
      }
      const length = end - start + 1;
      if (left >= right && length > bestLength) {
        bestStart = start;
        bestLength = length;
      }
    }
  }
  return characters.slice(bestStart, bestStart + bestLength).join("");
}
```

#### Visualization
```text
O(n^2) substrings x up to O(n) palindrome check
```
#### Dry Run
Candidates include `b`, `ba`, `bab` (valid), `baba` (invalid), and many others.
#### Brute-Force Complexity
`O(n²)` substrings each may take `O(n)` to check: `O(n³)` time and `O(n)` substring/output space.
### 8. Why Brute Force Is Not Ideal
The same mirrored comparisons are repeated across many generated substrings.
### 9. Optimal Approach
For the standard beginner interview solution, expand from every odd/even center and update only for a strictly longer palindrome, preserving the earliest tie. Manacher's algorithm is the advanced `O(n)` follow-up.
### 10. Visualization
```text
b a b a d
  ^ center a -> b-a-b expands length 3
    ^ center b -> a-b-a expands length 3, tie keeps bab
```
### 11. Step-by-Step Dry Run
Centers at indexes `0` and gaps yield length one or zero. Odd center `1` expands to `bab`; odd center `2` expands to `aba`, but the equal length does not replace the earlier answer.
### 12. TypeScript Solution
```typescript
function longestPalindromicSubstring(text: string): string {
  const characters = [...text];
  let bestStart = 0;
  let bestLength = 0;

  const expand = (leftStart: number, rightStart: number): void => {
    let left = leftStart;
    let right = rightStart;
    while (left >= 0 && right < characters.length && characters[left] === characters[right]) {
      const length = right - left + 1;
      if (length > bestLength) { bestStart = left; bestLength = length; }
      left -= 1;
      right += 1;
    }
  };

  for (let center = 0; center < characters.length; center += 1) {
    expand(center, center);
    expand(center, center + 1);
  }
  return characters.slice(bestStart, bestStart + bestLength).join("");
}
```
### 13. Code Walkthrough
Two expansion calls cover both center types. Strictly longer updates preserve the first palindrome among equal lengths because centers are processed left to right.
### 14. Complexity Analysis
**Time: `O(n²)`** worst case because `O(n)` centers may each expand `O(n)` positions. **Space: `O(n)`** for code points and returned slice; expansion state is `O(1)`.
### 15. Edge Cases
- Empty returns empty. - One symbol returns itself. - Even palindrome such as `abba` works. - All equal causes quadratic expansion.
### 16. Common Mistakes
- Checking only odd centers.
- Confusing substring with subsequence.
- Updating on equal length and breaking earliest-tie behavior.
- Calling center expansion linear.
### 17. Interview Follow-ups
1. Implement Manacher's `O(n)` algorithm. 2. Return all longest ties. 3. Count palindromic substrings.
### 18. Variations
- Longest palindromic subsequence. - Minimum palindrome cuts. - Palindrome radii.
### 19. Pattern Connections
Center expansion reuses local symmetry; dynamic programming and Manacher reuse broader palindrome state.
### 20. NestJS Project Connection
Expose `POST /api/dsa/strings/longest-palindrome`; trace center, expansion bounds, and best updates with strict size limits.
### 21. Mini Challenge
Return the number of all palindromic substrings by counting every successful center expansion.

---

# Topic Summary

## What You Learned
- Strings are immutable, so transformations allocate output or mutable character storage.
- JavaScript indexes UTF-16 code units; spreading iterates Unicode code points.
- Case, punctuation, whitespace, normalization, and alphabet rules belong in the contract.
- Two pointers compare mirrored characters and reverse mutable representations.
- Frequencies answer anagram and repeating/unique-character questions.
- Tokenization rules control word-level algorithms.
- Stacks validate nested delimiters.
- Sliding windows avoid rescanning overlapping substrings.
- Center expansion reduces longest-palindrome search from cubic to quadratic time.

## Patterns You Should Recognize
- character classification and accumulation
- inward two-pointer comparison
- frequency balance and stable frequency selection
- seen-set stable filtering
- tokenize, transform, and join
- vertical prefix comparison
- run-length traversal
- stack-based nesting validation
- variable sliding windows with last-seen indexes
- odd/even center expansion

## Complexity Cheat Sheet
| Operation or pattern | Time | Space |
|---|---:|---:|
| Traverse `n` code points | `O(n)` | `O(1)` state |
| Spread/join string | `O(n)` | `O(n)` |
| Character frequency map | Average `O(n)` | `O(u)` |
| Sort characters | `O(n log n)` | Implementation-dependent |
| KMP preprocessing and search | `O(n+m)` | `O(m)` plus representations |
| Stack delimiter validation | `O(n)` | `O(n)` |
| Last-seen sliding window | Average `O(n)` | `O(u)` plus representation |
| Center expansion palindrome | `O(n²)` | `O(1)` state plus output/representation |

## Common Interview Traps
- Leaving normalization and tokenization behavior undefined.
- Treating code units, code points, and grapheme clusters as identical.
- Claiming an immutable string was modified in place.
- Ignoring copying from spread, slice, concatenation, or join.
- Using a set where character frequencies matter.
- Returning the first duplicate event instead of earliest repeating first occurrence.
- Moving a sliding-window left boundary backward.
- Forgetting even palindrome centers.
- Assuming a built-in substring search has a specified asymptotic guarantee.

## Practice Checklist
- [ ] Problem 36 — Reverse a string
- [ ] Problem 37 — Check if a string is a palindrome
- [ ] Problem 38 — Check if two strings are anagrams
- [ ] Problem 39 — Find the first non-repeating character
- [ ] Problem 40 — Find the first repeating character
- [ ] Problem 41 — Count vowels and consonants
- [ ] Problem 42 — Remove duplicate characters
- [ ] Problem 43 — Reverse words in a string
- [ ] Problem 44 — Find the longest word in a string
- [ ] Problem 45 — Check if one string is a rotation of another
- [ ] Problem 46 — Find the longest common prefix
- [ ] Problem 47 — String compression
- [ ] Problem 48 — Valid Parentheses
- [ ] Problem 49 — Longest substring without repeating characters
- [ ] Problem 50 — Longest palindromic substring