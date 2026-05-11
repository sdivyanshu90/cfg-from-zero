## Chapter 4 — The CYK Parsing Algorithm

The Cocke-Younger-Kasami algorithm is a **bottom-up, chart-based** parser. It builds a triangular table of evidence, starting with individual tokens and combining spans into larger and larger structures. At the end, it checks whether the top cell contains the start symbol.

---

### The Core Idea

Every cell in the CYK table answers one question:

```
  table[i][j]  =  the set of non-terminals that can derive
                  the substring from token i to token j  (inclusive, 0-indexed)
```

If `table[0][n-1]` contains the start symbol `S`, the input is accepted.

---

### The Grammar (CNF Required)

We use a small CNF grammar to parse `a b a b`:

```
  S  →  A B | S S
  A  →  a
  B  →  b
```

Input tokens:  `a  b  a  b`   (indices 0, 1, 2, 3)

---

### Step-by-Step Table Filling

The table is a triangle. Cell `[i][j]` is only valid when `i ≤ j`. Fill the bottom diagonal first (span length 1), then span length 2, and so on upward.

#### Diagonal 0 — span length 1 (each single token)

Apply only **leaf rules** `A → a`:

```
  token index:    0        1        2        3
  token:          a        b        a        b
                  ─────────────────────────────
  table[i][i]:  { A }    { B }    { A }    { B }
```

#### Diagonal 1 — span length 2

For each cell `[i][i+1]`, try every split point `k = i`:

```
  Cell [0][1]  covers  a b
    Split k=0:  left=[0][0]={A},  right=[1][1]={B}
    Rule  S → A B  matches!
    → table[0][1] = { S }

  Cell [1][2]  covers  b a
    Split k=1:  left=[1][1]={B},  right=[2][2]={A}
    No rule has RHS (B, A)
    → table[1][2] = { }

  Cell [2][3]  covers  a b
    Split k=2:  left=[2][2]={A},  right=[3][3]={B}
    Rule  S → A B  matches!
    → table[2][3] = { S }
```

#### Diagonal 2 — span length 3

```
  Cell [0][2]  covers  a b a
    Split k=0:  left=[0][0]={A},  right=[1][2]={}   → no rules fire
    Split k=1:  left=[0][1]={S},  right=[2][2]={A}  → no rule has (S, A)
    → table[0][2] = { }

  Cell [1][3]  covers  b a b
    Split k=1:  left=[1][1]={B},  right=[2][3]={S}  → no rule has (B, S)
    Split k=2:  left=[1][2]={},   right=[3][3]={B}  → empty left, nothing fires
    → table[1][3] = { }
```

#### Diagonal 3 — span length 4 (full string)

```
  Cell [0][3]  covers  a b a b
    Split k=0:  left=[0][0]={A},  right=[1][3]={}   → nothing
    Split k=1:  left=[0][1]={S},  right=[2][3]={S}
                Rule  S → S S  matches!             ← found it
    Split k=2:  left=[0][2]={},   right=[3][3]={B}  → empty left
    → table[0][3] = { S }
```

---

### The Complete CYK Table

```
  j→    0        1        2        3
  i↓  ┌────────┬────────┬────────┬────────┐
   0  │ { A }  │ { S }  │ {    } │ { S }  │
      ├────────┼────────┼────────┼────────┤
   1  │        │ { B }  │ {    } │ {    } │
      ├────────┼────────┼────────┼────────┤
   2  │        │        │ { A }  │ { S }  │
      ├────────┼────────┼────────┼────────┤
   3  │        │        │        │ { B }  │
      └────────┴────────┴────────┴────────┘

  Start symbol S is in table[0][3]  → input ACCEPTED ✓
```

---

### Tracing the Parse Back to a Tree

The table tells you *that* a parse exists. To recover *which* parse, you store **back-pointers** while filling the table.

```
  table[0][3] contains S via:
    rule S → S S,  split at k=1
    ├── left:  table[0][1] contains S via: rule S → A B, split at k=0
    │           ├── left:  table[0][0] = A via: A → a
    │           └── right: table[1][1] = B via: B → b
    └── right: table[2][3] contains S via: rule S → A B, split at k=2
                ├── left:  table[2][2] = A via: A → a
                └── right: table[3][3] = B via: B → b
```

Reconstructed parse tree:

```
            S
           / \
          S   S
         / \ / \
        A  B A  B
        |  | |  |
        a  b a  b
```

---

### The Recurrence in One Place

```
  Base case (span length 1):
    table[i][i] += A    for every rule  A → tokens[i]

  Inductive case (span length L > 1):
    for each split point k  where  i ≤ k < j:
      for each rule  A → B C  in the grammar:
        if  B ∈ table[i][k]  and  C ∈ table[k+1][j]:
          table[i][j] += A

  Accept:
    S ∈ table[0][n-1]
```

---

### Complexity Analysis

```
  Table cells to fill:     O(n²)      — triangular, n = input length
  Split points per cell:   O(n)       — at most n-1 splits
  Rules to check:          O(|G|)     — |G| = number of grammar rules
  ─────────────────────────────────────────────────────────────────────
  Total time:              O(n³ · |G|)
```

The cubic factor `n³` comes from iterating over all spans `[i,j]` and all split points `k` inside each. For practical grammars and inputs of a few hundred tokens this is fast enough. For very long texts (thousands of tokens), more specialized algorithms (Earley, GLR) may be preferred.

---

### Why the Arrows Matter

Each arrow from a parent cell to two child cells is the proof that justified the parent. A filled cell without arrows is not a proof — it is just a claimed result. The arrows make the parse **verifiable**: you can trace any cell from the root all the way down to individual token recognitions.

---

### Common Failure Modes

```
  SYMPTOM                          LIKELY CAUSE
  ──────────────────────────────────────────────────────────────────
  Top cell is empty                Grammar not in CNF; or start symbol
                                   never appears in any rule's LHS

  Bottom row has empty cells       A token was not matched by any
                                   leaf rule A → token

  Middle cells all empty           No binary rule whose (B, C) pair
                                   appears in adjacent left/right cells

  Accepted when you expect reject  Tokenization mismatch — extra or
                                   missing tokens in the input stream
```

---

### Self-Check

```
  Question                                         Answer
  ──────────────────────────────────────────────────────────────────────
  What does table[i][j] store?                     All non-terminals that can
                                                   derive tokens[i..j].

  What fills the bottom diagonal?                  Leaf rules: A → single_token.

  What fills higher diagonals?                     Binary rules: A → B C where
                                                   B and C come from left/right
                                                   sub-cells.

  What is the acceptance condition?                S ∈ table[0][n-1].

  Why must the grammar be in CNF?                  CYK's recurrence assumes
                                                   exactly two children per
                                                   interior node.

  What is the time complexity?                     O(n³ · |G|).
```
