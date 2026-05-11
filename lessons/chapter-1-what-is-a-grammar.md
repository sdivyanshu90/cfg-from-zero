## Chapter 1 — What Is a Grammar?

A **grammar** is a finite set of rules that describes an infinite set of strings. You hand it a string and ask: "does this belong?" Or you run it in reverse and ask: "what strings can you produce?" Both directions are useful — parsers do the first, generators do the second.

---

### The Chomsky Hierarchy

Noam Chomsky classified all formal grammars into four nested classes. Each inner class is strictly less powerful but faster to parse.

```
         ┌──────────────────────────────────────────┐
         │  Type 0 — Unrestricted (Turing-complete)  │
         │  Rules: αAβ → γ  (anything goes)          │
         │  ┌────────────────────────────────────┐   │
         │  │  Type 1 — Context-Sensitive (CSG)  │   │
         │  │  Rules: αAβ → αγβ  (context stays) │   │
         │  │  ┌──────────────────────────────┐  │   │
         │  │  │  Type 2 — Context-Free (CFG) │  │   │
         │  │  │  Rules: A → γ  (A alone)     │  │   │
         │  │  │  ┌──────────────────────┐    │  │   │
         │  │  │  │  Type 3 — Regular    │    │  │   │
         │  │  │  │  Rules: A → aB | a   │    │  │   │
         │  │  │  └──────────────────────┘    │  │   │
         │  │  └──────────────────────────────┘  │   │
         │  └────────────────────────────────────┘   │
         └──────────────────────────────────────────┘
```

- **Regular grammars** describe patterns like email addresses or identifiers. A regex engine is sufficient.
- **Context-free grammars** add balanced nesting (parentheses, HTML tags, code blocks). Regex cannot handle this.
- **Context-sensitive grammars** allow rules that look at surrounding context. Natural language lives here.
- **Unrestricted grammars** are equivalent to Turing machines. Parsing is undecidable in general.

> **Why do we stop at Type 2?** Because CFGs hit the sweet spot: expressive enough for programming languages and structured data, but tractable enough to parse efficiently (polynomial time).

---

### Formal Definition of a CFG

A context-free grammar is a 4-tuple **G = (N, T, P, S)** where:

```
  ┌─────┬───────────────────────────────────────────────────────────────┐
  │  N  │  Non-terminals — a finite set of placeholder symbols          │
  │     │  Example: { E, T, F }                                         │
  ├─────┼───────────────────────────────────────────────────────────────┤
  │  T  │  Terminals — a finite set of actual output symbols            │
  │     │  Example: { id, +, *, (, ) }                                  │
  ├─────┼───────────────────────────────────────────────────────────────┤
  │  P  │  Productions — a finite set of rewrite rules A → α           │
  │     │  where A ∈ N and α ∈ (N ∪ T)*                               │
  ├─────┼───────────────────────────────────────────────────────────────┤
  │  S  │  Start symbol — one distinguished member of N                 │
  │     │  Every derivation begins here                                 │
  └─────┴───────────────────────────────────────────────────────────────┘
```

**The key constraint that makes it "context-free":** every production has exactly one non-terminal on the left-hand side, with nothing beside it. No surrounding context is required before you can apply the rule.

Compare these two rules:

```
  Context-free:    E  →  E + T          (apply whenever you see E)
  Context-sensitive:  a E b  →  a T b   (apply only when E is between a and b)
```

---

### Anatomy of the Arithmetic Expression Grammar

This grammar recognizes expressions like `id + id * id` and `(id + id) * id`:

```
  Production rules (P)
  ────────────────────
  E  →  E + T         rule 1
  E  →  T             rule 2
  T  →  T * F         rule 3
  T  →  F             rule 4
  F  →  ( E )         rule 5
  F  →  id            rule 6

  Non-terminals (N): { E, T, F }
  Terminals     (T): { id, +, *, (, ) }
  Start symbol  (S): E
```

Notice the layered structure: `E` delegates multiplication precedence to `T`, and `T` delegates atoms and parenthesized groups to `F`. This layering **encodes operator precedence directly in the grammar shape**.

---

### Terminals vs Non-Terminals: The Core Distinction

```
  SYMBOL                 CAN BE EXPANDED?   APPEARS IN FINAL STRING?   TYPE
  ─────────────────────────────────────────────────────────────────────────
  E (expression)         ✓  Yes             ✗  No                       Non-terminal
  T (term)               ✓  Yes             ✗  No                       Non-terminal
  F (factor)             ✓  Yes             ✗  No                       Non-terminal
  id                     ✗  No              ✓  Yes                      Terminal
  +                      ✗  No              ✓  Yes                      Terminal
  *                      ✗  No              ✓  Yes                      Terminal
  (                      ✗  No              ✓  Yes                      Terminal
  )                      ✗  No              ✓  Yes                      Terminal
```

> **Common trap:** uppercase letter does NOT automatically mean non-terminal. It is only a convention. The real test is structural: does this symbol appear on the left-hand side of any rule? If yes, it is a non-terminal. If no, it is a terminal.

---

### The Factory Mental Model

Imagine a small assembly line:

```
  [E] ──rule 1──▶ [E] + [T]          unfinished parts waiting on the belt
       ──rule 2──▶ [T]

  [T] ──rule 3──▶ [T] * [F]
       ──rule 4──▶ [F]

  [F] ──rule 5──▶ ( [E] )
       ──rule 6──▶ id                 ◀── finished part, ready to ship

  ┌────────────────┬────────────────────────────────────────────┐
  │  Conveyor belt │  Non-terminals rolling through             │
  │  Finished bin  │  Terminals — nothing left to do to them    │
  │  Machine rules │  Productions — the rewrite settings        │
  │  First part    │  Start symbol — E is always the first drop │
  └────────────────┴────────────────────────────────────────────┘
```

Parsing is "reverse manufacturing": given a finished product (the input string), work backward and figure out which machine settings could have produced it.

---

### Syntax vs Semantics

A grammar only gives you **syntax** — the shape of legal strings. It says nothing about meaning.

```
  String: id + id * id
           │         │
      grammar says:  grammar says:
      "this is       "this has
       legal"         structure"

  But which id means what? That is SEMANTICS —
  handled later by an evaluator, type-checker, or compiler.
```

The grammar for arithmetic says `id + id * id` is legal and that `*` binds tighter than `+` (because `T` is deeper than `E`). What the identifiers *refer to* is outside the grammar's scope.

---

### Inline vs Block Rules: Shorthand Notation

Grammars often abbreviate multiple alternatives with a pipe `|`:

```
  Verbose form:                 Shorthand form:
  E → E + T                     E → E + T | T
  E → T

  T → T * F                     T → T * F | F
  T → F

  F → ( E )                     F → ( E ) | id
  F → id
```

Both notations describe exactly the same grammar. The shorthand is just easier to read and write.

---

### Self-Check

Before moving on, you should be able to answer these questions by inspecting the grammar:

```
  Question                                    Answer
  ─────────────────────────────────────────────────────────────
  How many non-terminals are there?           3  (E, T, F)
  How many terminals are there?               5  (id, +, *, (, ))
  How many production rules are there?        6
  What is the start symbol?                   E
  Which symbol can generate the empty string? None in this grammar
  Which rule handles grouping with parens?    F → ( E )
```
