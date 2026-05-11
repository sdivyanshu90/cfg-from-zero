## Chapter 2 — Derivations and Parse Trees

A derivation is a **proof** that a string belongs to a grammar's language. A parse tree is the **diagram** of that proof. Both are essential: the derivation shows the sequence of moves; the tree shows the structure those moves built.

---

### What Is a Derivation?

Starting from the start symbol, you repeatedly replace one non-terminal with the right-hand side of one of its rules. Each intermediate string is called a **sentential form**. You stop when no non-terminals remain.

```
  Grammar:
    E → E + T | T
    T → T * F | F
    F → ( E ) | id

  Deriving  id + id * id  from  E:

  Step   Sentential form       Rule applied
  ─────────────────────────────────────────────────────
   0     E                     (start)
   1     E + T                 E → E + T
   2     T + T                 E → T           (left E)
   3     F + T                 T → F
   4     id + T                F → id
   5     id + T * F            T → T * F
   6     id + F * F            T → F           (left T)
   7     id + id * F           F → id
   8     id + id * id          F → id          (done ✓)
```

Each arrow is a **rewrite step** — one non-terminal replaced by one rule's right-hand side. The derivation is a sequence, not a branching structure.

---

### Leftmost vs Rightmost Derivations

You have a choice at every step: which non-terminal do you expand next? Two canonical strategies exist:

```
  LEFTMOST derivation: always expand the leftmost non-terminal first.
  RIGHTMOST derivation: always expand the rightmost non-terminal first.
```

Both produce the same final string and the same parse tree — they just visit the tree nodes in a different order.

#### Leftmost derivation of  id + id * id

```
  E
  ⇒  E + T          (E → E + T,    expand leftmost E)
  ⇒  T + T          (E → T,        expand leftmost E)
  ⇒  F + T          (T → F,        expand leftmost T)
  ⇒  id + T         (F → id,       expand leftmost F)
  ⇒  id + T * F     (T → T * F,    expand leftmost T)
  ⇒  id + F * F     (T → F,        expand leftmost T)
  ⇒  id + id * F    (F → id,       expand leftmost F)
  ⇒  id + id * id   (F → id,       expand leftmost F)  ✓
```

#### Rightmost derivation of  id + id * id

```
  E
  ⇒  E + T          (E → E + T,    expand rightmost E)
  ⇒  E + T * F      (T → T * F,    expand rightmost T)
  ⇒  E + T * id     (F → id,       expand rightmost F)
  ⇒  E + F * id     (T → F,        expand rightmost T)
  ⇒  E + id * id    (F → id,       expand rightmost F)
  ⇒  T + id * id    (E → T,        expand rightmost E)
  ⇒  F + id * id    (T → F,        expand rightmost T)
  ⇒  id + id * id   (F → id,       expand rightmost F)  ✓
```

> The steps look different but produce the **identical parse tree**. Derivation order is just a traversal strategy.

---

### The Parse Tree

The parse tree (also called a **concrete syntax tree**) records every decision made during the derivation. Interior nodes are non-terminals; leaves are terminals.

#### Parse tree for  id + id * id  (unambiguous grammar)

```
            E
           /|\
          E + T
          |   |\
          T   T * F
          |   |   |
          F   F   id
          |   |
          id  id

  Reading:
    E  ──▶  E + T         (addition at the top level)
    E  ──▶  T             (left operand is a single term)
    T  ──▶  F             (left term is a single factor)
    F  ──▶  id            (leaf: first id)
    T  ──▶  T * F         (right operand is a product)
    T  ──▶  F             (left side of product)
    F  ──▶  id            (leaf: second id)
    F  ──▶  id            (leaf: third id)
```

The tree structure shows that `*` has **higher precedence** than `+` — the multiplication subtree is deeper, meaning it binds before the addition.

---

### Ambiguity: Two Trees for One String

Consider this simpler (but broken) grammar that drops the precedence layering:

```
  E → E + E | E * E | ( E ) | id     ← ambiguous!
```

The string `id + id * id` can be parsed in **two different ways**:

#### Tree 1 — multiply first (correct semantics)

```
        E
       /|\
      E * E
      |   |
      E   id
     /|\
    E + E
    |   |
    id  id

  Meaning: id + (id * id)
```

#### Tree 2 — add first (wrong semantics for normal arithmetic)

```
        E
       /|\
      E + E
      |   |
      id  E
         /|\
        E * E
        |   |
        id  id

  Meaning: (id + id) * id
```

> Same string. Two trees. Two different meanings. **That is genuine ambiguity** — not just a different expansion order.

#### How to distinguish ambiguity from different derivation order

```
  Same tree + different expansion order   →  NOT ambiguous  (just a traversal choice)
  Different trees                         →  AMBIGUOUS      (the grammar is broken)
```

The fix is to encode precedence in the grammar structure, exactly as the E/T/F grammar does.

---

### Sentential Forms as Snapshots

At every step in a derivation, the current string is called a **sentential form**. You can think of it as a snapshot of what the parse tree looks like if you read its frontier left to right.

```
  Derivation step             Frontier of partial tree
  ─────────────────────────────────────────────────────
  E                           [E]
  E + T                       [E] [+] [T]
  T + T                       [T] [+] [T]
  F + T                       [F] [+] [T]
  id + T                      id  [+] [T]
  id + T * F                  id  [+] [T] [*] [F]
  id + F * F                  id  [+] [F] [*] [F]
  id + id * F                 id   +   id  [*] [F]
  id + id * id                id   +   id   *   id

  Boxed = non-terminal (still unresolved)
  Plain = terminal (locked in)
```

The final row is all terminals — the string is complete and every tree node has been assigned.

---

### What Happens With Parentheses

The rule `F → ( E )` resets the precedence level. A parenthesized group starts a fresh `E`, so `(id + id) * id` correctly multiplies the sum:

```
          E
         /|\
        T * F
        |   |
        F  id
        |
      ( E )
       /|\
      E + T
      |   |
      T   F
      |   |
      F   id
      |
      id

  Meaning: (id + id) * id   ← parens force the addition to be deeper
```

Parentheses don't need special precedence rules in the grammar — the recursive call to `E` inside `F` handles them automatically.

---

### Reading the Stepper Visualization

The interactive stepper highlights which symbols in the current sentential form are still unresolved (non-terminals). When you advance a step:

- One highlighted chip disappears from the frontier.
- New chips appear in its place (the RHS of the chosen rule).
- The matching tree node expands with new children at the same moment.

Watch where the split point around `+` or `*` appears in the tree. That split point reveals the effective precedence the grammar assigns.

---

### Self-Check

```
  Question                                          Answer
  ─────────────────────────────────────────────────────────────────────────
  Can two leftmost derivations produce the          Yes — only if the grammar
  same string but different trees?                  is ambiguous.

  Does leftmost vs rightmost derivation affect      No — both yield the same tree.
  the parse tree?

  What does it mean for a grammar to be            Some string has two or more
  ambiguous?                                        distinct parse trees.

  Which rule is responsible for operator            The layered non-terminal
  precedence in the E/T/F grammar?                  structure: E → T → F.

  How do parentheses restore lower precedence?      F → ( E ) restarts at E,
                                                    so the interior can grow freely.
```
