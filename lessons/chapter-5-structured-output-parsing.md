## Chapter 5 — Grammar-Constrained Generation

Up to now, a grammar has been something you hand to a **parser** after a string exists. This chapter flips the direction: a grammar becomes something a **language model** consults *before* each token is emitted, so the output is guaranteed to be valid before it even finishes generating.

---

### The Two Directions of a Grammar

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  PARSING (backward)                                             │
  │  Input:   a finished string                                     │
  │  Question: is this string in the language?                      │
  │  Direction: string → tree (left to right, bottom-up)           │
  └─────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │  CONSTRAINED GENERATION (forward)                               │
  │  Input:   a partial prefix already emitted                      │
  │  Question: which tokens can come next?                          │
  │  Direction: prefix → allowed next tokens (proactive)            │
  └─────────────────────────────────────────────────────────────────┘
```

The key insight: if you can enumerate all strings in the language that **start with** the current prefix, the next token must be the first character that differs — anything else creates a string that cannot be completed.

---

### Prefix Filtering Step by Step

Suppose the grammar describes valid JSON objects and the model has so far emitted:

```
  { "name":
```

At this point, the only valid continuations are things that can follow `"name":` in JSON. Every other token would immediately make the string unrecoverable.

```
  All valid JSON strings
       │
       ▼  filter: must start with  { "name":
       │
  { "name": "alice" }
  { "name": "bob",  "age": 30 }
  { "name": null }
  { "name": 42 }
  { "name": [ ... ] }
       │
       ▼  collect first unseen character (the token right after the prefix)
       │
  Allowed next tokens:  "  null  0-9  [  {
  Forbidden:            }  ,  :  whitespace that does not lead anywhere  ...
```

---

### The Token Mask

A **logit mask** is applied to the language model's output distribution before sampling. Forbidden tokens get their logits set to −∞ (effectively zero probability).

```
  Logits before masking:
  ┌──────────┬────────┬──────────────┐
  │  token   │ logit  │  status      │
  ├──────────┼────────┼──────────────┤
  │   "      │  3.2   │  allowed  ✓  │
  │   null   │  1.8   │  allowed  ✓  │
  │   42     │  2.1   │  allowed  ✓  │
  │   }      │  0.9   │  forbidden ✗ │
  │   ,      │  0.4   │  forbidden ✗ │
  │   hello  │  1.1   │  forbidden ✗ │
  └──────────┴────────┴──────────────┘

  After masking:
  ┌──────────┬────────┬──────────────┐
  │   "      │  3.2   │  sampled     │
  │   null   │  1.8   │  sampled     │
  │   42     │  2.1   │  sampled     │
  │   }      │  -∞    │  blocked     │
  │   ,      │  -∞    │  blocked     │
  │   hello  │  -∞    │  blocked     │
  └──────────┴────────┴──────────────┘
```

The model still picks the most likely token among the allowed ones — it just cannot pick something structurally illegal.

---

### Decoder Sketch in JavaScript

```js
// candidates = all valid strings the grammar can produce
// prefix = what has been emitted so far

function allowedNextTokens(candidates, prefix) {
  const surviving = candidates.filter(s => s.startsWith(prefix));
  const nextChars = new Set(
    surviving
      .map(s => s[prefix.length])
      .filter(c => c !== undefined)
  );
  return nextChars;
}

function maskLogits(logits, tokenVocab, allowedSet) {
  return logits.map((logit, i) => {
    const token = tokenVocab[i];
    return allowedSet.has(token) ? logit : -Infinity;
  });
}
```

In production systems, `candidates` is not enumerated explicitly — instead, an incremental parser runs in lockstep with the decoder and exposes a `canContinue(prefix + nextToken)` predicate.

---

### The Allowed-Set Shrinks as the Prefix Grows

```
  Prefix: ""
  Allowed:  {  [  "  0-9  null  true  false  ← almost anything

  Prefix: "{"
  Allowed:  "  }  (string key or close brace)

  Prefix: "{ "
  Allowed:  "  }

  Prefix: '{ "name"'
  Allowed:  :

  Prefix: '{ "name":'
  Allowed:  "  0-9  null  true  false  {  [

  Prefix: '{ "name": "alice"'
  Allowed:  ,  }

  Prefix: '{ "name": "alice" }'
  Allowed:  (end of string — generation must stop here)
```

Each new token narrows the allowed set. The grammar acts as a **running parse state** that the decoder checks before every sample.

---

### Incremental Parsing State Machine

For efficient real-time masking, systems pre-compile the grammar into a **pushdown automaton** or use an Earley-style incremental recognizer. At each step, the current state encodes exactly which grammar rules are partially matched.

```
  Grammar state after prefix  { "name":

  Parser is in state:
  ┌─────────────────────────────────────┐
  │  object  →  { pairs               ← waiting for a value
  │  pairs   →  pair  ·  more-pairs   ← dot shows how far we are
  │  pair    →  string  :  ·  value   ← about to parse value
  │  value   →  string  |  number  |  null  |  bool  |  array  |  object
  └─────────────────────────────────────┘

  The set of rules with the dot before a terminal tells you exactly
  which tokens are currently allowed.
```

---

### Real-World Systems

```
  LIBRARY / TOOL      GRAMMAR FORMAT         NOTES
  ────────────────────────────────────────────────────────────────────
  Outlines            Pydantic / Regex        Compiles schema to FSM
  Guidance            Handlebars-style DSL    Interleaves control flow
  LMQL                SQL-like syntax         Query language for LLMs
  llama.cpp (GBNF)    Extended BNF            Native C++ integration
  vllm                Outlines backend        Batched GPU inference
```

All of them solve the same core problem: convert a grammar or schema into an efficient **next-token predicate** that runs at decode time.

---

### Why This Matters for LLM Applications

Without grammar constraints, a language model generating JSON will occasionally:
- forget to close a bracket,
- emit a key without a value,
- produce `undefined` instead of `null`,
- truncate mid-string.

With grammar constraints, these failures are **structurally impossible** — the model cannot emit a token that would make the output unrecoverable, no matter how confused its probability distribution is.

```
  Without constraints:   { "name": "alice", "age":    ← model stops mid-object
  With constraints:      { "name": "alice", "age": 30 }  ← forced to complete
```

---

### Key Connections Back to Previous Chapters

```
  Chapter 1 — Grammar as a set of rules
    ↓  those rules define which strings are valid
  Chapter 2 — Derivations trace how strings are built
    ↓  running a derivation forward = generating; backward = parsing
  Chapter 4 — CYK fills a table of partial parse evidence
    ↓  incremental parsing uses the same idea per-token instead of per-span
  Chapter 5 — Grammar constrains the decoder token by token
    ↓  allowed set = non-terminals that can still be satisfied given the prefix
```

---

### Self-Check

```
  Question                                         Answer
  ──────────────────────────────────────────────────────────────────────
  What does the token mask do?                     Sets logits of forbidden
                                                   tokens to -∞ before sampling.

  Why does the allowed set shrink over time?       Each emitted token commits
                                                   to one parse path; others
                                                   are eliminated.

  What is the role of the grammar at decode time?  It defines which tokens can
                                                   still lead to a valid string.

  How is "valid continuation" defined?             There exists at least one
                                                   string in the language that
                                                   extends the current prefix.

  Name one production system that uses this.       Outlines, Guidance, GBNF, LMQL.
```
