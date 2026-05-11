## Chapter 3 — Chomsky Normal Form

Before the CYK algorithm can run, every rule in the grammar must have one of exactly two shapes. This chapter is about the **surgery** that transforms any CFG into that restricted form without changing the language it accepts.

---

### What CNF Requires

A grammar is in **Chomsky Normal Form (CNF)** if every production has one of these two forms:

```
  A  →  B C          interior rule: exactly two non-terminal children
  A  →  a            leaf rule: exactly one terminal

  (Plus a special case: S → ε is allowed only if ε is in the language.)
```

Every other shape is **forbidden** in CNF:

```
  FORBIDDEN                   WHY IT IS FORBIDDEN
  ────────────────────────────────────────────────────────────────────
  A  →  ε                     empty right-hand side (epsilon production)
  A  →  B                     unit production — single non-terminal RHS
  A  →  B C D                 too many children — must binarize
  A  →  a B                   mixed terminal and non-terminal in long rule
  A  →  B a C                 same — terminal inside longer rule
```

---

### Why CNF Makes CYK Possible

CYK fills a 2-D table where each cell `[i,j]` asks: "which non-terminals can generate the substring from token `i` to token `j`?" The recurrence needs every interior decision to split a span into exactly **two** sub-spans. CNF guarantees that. Without it, the split could have three or more sub-spans and the recurrence breaks.

```
  CNF rule  A → B C  maps to:
  ┌─────────────────────────┐
  │           A             │  span [i, j]
  │          / \            │
  │         B   C           │
  │      [i,k] [k+1,j]      │  exactly one split point k
  └─────────────────────────┘

  If the rule were A → B C D, there would be two split points and
  the table cell would need to consider O(n²) pairs instead of O(n).
```

---

### The Five-Step Transformation Pipeline

The pipeline runs in order. Skipping or reordering steps can introduce new problems. Each step conserves the language (same strings accepted before and after).

```
  Original grammar
       │
       ▼  Step 1: Remove epsilon-productions
       │
       ▼  Step 2: Remove unit-productions
       │
       ▼  Step 3: Remove useless symbols
       │
       ▼  Step 4: Binarize long rules
       │
       ▼  Step 5: Isolate terminals in long rules
       │
  CNF grammar
```

---

### Step 1 — Remove Epsilon-Productions

An epsilon-production has the form `A → ε`. Find every non-terminal that can derive ε (directly or through a chain), then for each rule containing such a symbol, add a new version of that rule with the nullable symbol optionally omitted.

#### Before

```
  S  →  A B
  A  →  a A | ε
  B  →  b
```

#### Finding nullable symbols

```
  A is directly nullable:    A → ε  ✓
  S is nullable?             S → A B, and A is nullable but B is not, so S → B only
                             S is not nullable.
```

#### After (epsilon-productions removed)

```
  S  →  A B | B              (version without A, since A can vanish)
  A  →  a A | a              (version without the trailing A)
  B  →  b
```

> The empty string `ε` is no longer directly derivable via a rule, but the language is preserved: any string that was derivable before is still derivable now.

---

### Step 2 — Remove Unit-Productions

A unit-production has the form `A → B` (a single non-terminal on the RHS). These create invisible chains that waste steps and confuse the CNF structure.

#### Before

```
  E  →  E + T | T
  T  →  T * F | F
  F  →  ( E ) | id
```

#### Unit-production chains

```
  E  ─unit─▶  T  ─unit─▶  F
```

#### After (units eliminated by inlining)

```
  E  →  E + T | T * F | ( E ) | id
  T  →  T * F | ( E ) | id
  F  →  ( E ) | id
```

Each unit chain `A → B → ... → X` is replaced by copying all non-unit rules of the final destination `X` directly into `A`.

---

### Step 3 — Remove Useless Symbols

A symbol is **useless** if it is either:
- **Non-generating**: can never derive a string of terminals through any sequence of rules.
- **Unreachable**: can never be reached from the start symbol.

#### Before (contrived example)

```
  S  →  A b | D
  A  →  a
  D  →  D d           ← D can never produce a terminal string (loops forever)
  B  →  b             ← B is never referenced from S, A, or D
```

#### After

```
  S  →  A b
  A  →  a

  Removed:  D (non-generating), B (unreachable)
```

Always check generating status first, then reachability. Reachability analysis on a grammar containing non-generating symbols can give wrong results.

---

### Step 4 — Binarize Long Rules

Any rule with three or more symbols on the right-hand side is broken into a chain of binary rules using fresh helper non-terminals.

#### Before

```
  A  →  B C D E
```

#### After (binarized)

```
  A      →  B  BIN_1
  BIN_1  →  C  BIN_2
  BIN_2  →  D  E
```

The new symbols `BIN_1`, `BIN_2` are **scaffolding** — they exist only to encode the original four-symbol rule as a right-branching binary tree:

```
       A
      / \
     B  BIN_1
         / \
        C  BIN_2
            / \
           D   E
```

The language does not change: the only string `A` can now derive via this chain is still `B C D E` (as strings of terminals once all are expanded).

---

### Step 5 — Isolate Terminals in Long Rules

After binarization, some binary rules may still mix terminals and non-terminals:

```
  A  →  a B          ← a (terminal) next to B (non-terminal) — forbidden in CNF
```

Fix this by wrapping each terminal in a new unit non-terminal:

```
  A       →  TERM_a  B
  TERM_a  →  a
```

#### Full before/after for a mixed long rule

```
  Before:   A  →  a B c D

  After binarization:
    A      →  a  BIN_1
    BIN_1  →  B  BIN_2
    BIN_2  →  c  D

  After terminal isolation:
    A       →  TERM_a  BIN_1
    BIN_1   →  B       BIN_2
    BIN_2   →  TERM_c  D
    TERM_a  →  a
    TERM_c  →  c
```

---

### Worked Example: Full Pipeline

Start with a small grammar:

```
  S  →  a S b | a b
```

This accepts strings like `ab`, `aabb`, `aaabbb`, etc.

#### After Step 1 (epsilon removal — none here, skip)

No change.

#### After Step 2 (unit removal — none here, skip)

No change.

#### After Step 3 (useless symbols — none here, skip)

No change.

#### After Step 4 (binarize)

`S → a S b` has length 3 — binarize:

```
  S     →  a  BIN_1
  BIN_1 →  S  b
  S     →  a  b
```

#### After Step 5 (isolate terminals)

`S → a BIN_1` and `BIN_1 → S b` and `S → a b` all mix terminals and non-terminals:

```
  S       →  TERM_a  BIN_1
  BIN_1   →  S       TERM_b
  S       →  TERM_a  TERM_b
  TERM_a  →  a
  TERM_b  →  b
```

This is valid CNF. Every rule is either `A → B C` or `A → a`. The language `{aⁿbⁿ | n ≥ 1}` is preserved.

---

### Color Legend for the Diff View

```
  AMBER   — An existing non-terminal was changed by this step
  TEAL    — A new helper non-terminal was introduced (BIN_*, TERM_*)
  CORAL   — A rule was deleted because its form is no longer allowed
```

---

### Auditing the Transformation

After any step, ask three questions:

```
  1. Shape check:  Do all rules now satisfy the expected form for this step?
  2. Helper check: Does every new BIN_* or TERM_* symbol have exactly one obvious job?
  3. Language check: Can the new grammar still derive the examples I care about?
```

If question 3 fails, a step was applied incorrectly. The most common error is forgetting to add all combinations when removing a nullable symbol from a rule.

---

### Self-Check

```
  Question                                         Answer
  ──────────────────────────────────────────────────────────────────────
  What two rule shapes does CNF allow?             A → B C  and  A → a

  Why must epsilon-productions be removed first?   Later steps can re-introduce
                                                   them if done out of order.

  What is a unit-production?                       A → B (single non-terminal RHS)

  Why are useless symbols removed?                 They add noise and can cause
                                                   the algorithm to loop or
                                                   produce wrong results.

  What is binarization?                            Splitting A → B C D into a
                                                   chain of binary rules.

  Does CNF change the language?                    No — same strings, different
                                                   rule shapes.
```
