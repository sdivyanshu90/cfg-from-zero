## Chapter 6 — Build Your Own Mini Parser

This final chapter has two halves. The first builds a **recursive descent parser** by hand — one function per non-terminal, written top-down. The second implements a **CYK table-filling parser** — algorithm-driven, bottom-up. Both parse the same language; they just think about the problem from opposite ends.

---

### Approach 1 — Recursive Descent

A recursive descent parser mirrors the grammar with functions. Each non-terminal becomes a function that consumes tokens and returns a parse node (or throws if the grammar rule cannot match).

#### Grammar for simple arithmetic

```
  E  →  T (('+' | '-') T)*
  T  →  F (('*' | '/') F)*
  F  →  '(' E ')' | NUMBER
```

#### Parser structure — one function per rule

```
  parseE()
    calls parseTerm(), then loops over + and -

  parseT()  (called parseTerm inside parseE)
    calls parseFactor(), then loops over * and /

  parseF()  (called parseFactor inside parseT)
    either matches ( E ) recursively,
    or matches a NUMBER literal
```

#### Full annotated implementation

```js
// Tokenizer produces an array like:
// [ {type:'NUM', val:3}, {type:'PLUS'}, {type:'NUM', val:4}, ... ]

let tokens = [];
let pos = 0;

function peek()    { return tokens[pos]; }
function consume() { return tokens[pos++]; }
function expect(type) {
  const tok = consume();
  if (tok.type !== type) throw new Error(`Expected ${type}, got ${tok.type}`);
  return tok;
}

// E → T (('+' | '-') T)*
function parseE() {
  let left = parseT();             // parse the first term
  while (peek()?.type === 'PLUS' || peek()?.type === 'MINUS') {
    const op = consume();          // consume the operator
    const right = parseT();        // parse the next term
    left = { op: op.type, left, right };   // build a node
  }
  return left;
}

// T → F (('*' | '/') F)*
function parseT() {
  let left = parseF();
  while (peek()?.type === 'STAR' || peek()?.type === 'SLASH') {
    const op = consume();
    const right = parseF();
    left = { op: op.type, left, right };
  }
  return left;
}

// F → '(' E ')' | NUMBER
function parseF() {
  if (peek()?.type === 'LPAREN') {
    consume();                     // eat the (
    const inner = parseE();        // recurse into expression
    expect('RPAREN');              // eat the )
    return inner;
  }
  const tok = expect('NUM');
  return { num: tok.val };         // leaf node
}
```

---

### Call Stack Trace for  3 + 4 * 2

```
  Input tokens:  NUM(3)  PLUS  NUM(4)  STAR  NUM(2)

  parseE()
  │  parseT()                       ← parse the first term
  │  │  parseF()
  │  │  │  consume NUM(3)  → { num: 3 }
  │  │  └─ return { num: 3 }
  │  │  peek = PLUS  → not * or /  → exit loop
  │  └─ return { num: 3 }
  │
  │  peek = PLUS  → enter loop
  │  consume PLUS
  │
  │  parseT()                       ← parse the right-hand term
  │  │  parseF()
  │  │  │  consume NUM(4)  → { num: 4 }
  │  │  └─ return { num: 4 }
  │  │  peek = STAR  → enter loop
  │  │  consume STAR
  │  │  parseF()
  │  │  │  consume NUM(2)  → { num: 2 }
  │  │  └─ return { num: 2 }
  │  │  peek = undefined → exit loop
  │  └─ return { op: STAR, left: {num:4}, right: {num:2} }
  │
  │  left = { op: PLUS,
  │            left:  { num: 3 },
  │            right: { op: STAR, left: {num:4}, right: {num:2} } }
  │  peek = undefined → exit loop
  └─ return the node above
```

The resulting AST shows `*` binding tighter than `+`, just like in the grammar:

```
       PLUS
      /    \
   num:3   STAR
           /  \
        num:4  num:2
```

---

### Evaluating the AST

Once you have the tree, a simple evaluator walks it:

```js
function evaluate(node) {
  if ('num' in node) return node.num;
  const l = evaluate(node.left);
  const r = evaluate(node.right);
  if (node.op === 'PLUS')  return l + r;
  if (node.op === 'MINUS') return l - r;
  if (node.op === 'STAR')  return l * r;
  if (node.op === 'SLASH') return l / r;
}

evaluate(ast);  // → 11   (3 + 4*2 = 3 + 8 = 11)
```

The parser gives you structure; the evaluator gives you meaning. Syntax then semantics — same order as the grammar hierarchy in Chapter 1.

---

### Approach 2 — CYK Builder Lab

Now build the same parser from the other direction. Start with the grammar, convert it to CNF (Chapter 3), then fill the CYK table (Chapter 4).

#### Grammar for the lab (accepts  ab, aabb, aaabbb, ...)

```
  S  →  A B | A S B     ← this is NOT in CNF yet

  Convert to CNF:
    Step 4 (binarize  A S B):
      S      →  A  BIN_1
      BIN_1  →  S  B
    Step 5 (isolate terminals — none needed, A and B already handle a/b):
      A      →  a
      B      →  b

  Final CNF grammar:
    S      →  A  B
    S      →  A  BIN_1
    BIN_1  →  S  B
    A      →  a
    B      →  b
```

#### CYK table for input  a a b b

```
  Tokens:  a  a  b  b   (indices 0 1 2 3)

  Span 1 (diagonal 0):
    [0][0] = { A }     rule A → a
    [1][1] = { A }     rule A → a
    [2][2] = { B }     rule B → b
    [3][3] = { B }     rule B → b

  Span 2 (diagonal 1):
    [0][1]: left={A} right={A}  → no rule with (A,A)     = {}
    [1][2]: left={A} right={B}  → S → A B  fires!        = { S }
    [2][3]: left={B} right={B}  → no rule with (B,B)     = {}

  Span 3 (diagonal 2):
    [0][2]: split k=0  left=[0][0]={A}  right=[1][2]={S}
              → no rule with (A,S) ... wait!  S → A BIN_1 needs BIN_1
            split k=1  left=[0][1]={}   right=[2][2]={B}
              → empty left, skip
            = {}

    [1][3]: split k=1  left=[1][1]={A}  right=[2][3]={}   → skip
            split k=2  left=[1][2]={S}  right=[3][3]={B}
              → BIN_1 → S B  fires!                       = { BIN_1 }

  Span 4 (diagonal 3):
    [0][3]: split k=0  left=[0][0]={A}  right=[1][3]={BIN_1}
              → S → A BIN_1  fires!                       = { S }
            (other splits also checked but this one wins)

  Accept:  S ∈ table[0][3]  →  ACCEPTED ✓
```

#### Recovered parse tree

```
          S
         / \
        A  BIN_1
        |   / \
        a  S   B
            / \ |
           A  B b
           |  |
           a  b
```

Reading: `a(a b)b` = `a + (inner aabb) + b` wait — actually `a(ab)b` = `aabb`. The inner `S` covers `ab` at positions 1-2.

---

### Side-by-Side Comparison

```
  FEATURE             RECURSIVE DESCENT          CYK
  ─────────────────────────────────────────────────────────────────
  Direction           Top-down (root → leaves)   Bottom-up (leaves → root)
  State               Call stack                 Table cells
  Grammar format      Any (written naturally)    Must be in CNF
  Handles ambiguity   First match only (usually) All parses in one pass
  Error messages      Easy — cursor shows where  Harder to localize
  Modify at runtime   Hard                       Change the table
  Typical use         Hand-written parsers        NLP / general CFGs
```

---

### Random String Generator

Running the grammar **forward** generates random strings in the language. At each non-terminal, pick a production at random and expand it. Stop when only terminals remain.

```js
function expand(symbol, grammar) {
  if (isTerminal(symbol)) return [symbol];

  // grammar[symbol] is an array of alternatives,
  // each alternative is an array of symbols
  const rhs = grammar[symbol][Math.floor(Math.random() * grammar[symbol].length)];
  return rhs.flatMap(s => expand(s, grammar));
}

const grammar = {
  E: [['E', '+', 'T'], ['T']],
  T: [['T', '*', 'F'], ['F']],
  F: [['(', 'E', ')'], ['id']],
};

expand('E', grammar).join(' ');
// Possible outputs:  id
//                    id + id
//                    id * id + id
//                    ( id + id ) * id
//                    id + id + id + id   (via repeated E → E + T)
```

Generation and parsing are inverses: generation explores the grammar forward; parsing explores it backward. The grammar is the shared specification that makes both work.

---

### Exit Skills Checklist

After this chapter you should be able to:

```
  ✓  Describe a grammar as a 4-tuple (N, T, P, S)
  ✓  Trace a leftmost derivation and draw the parse tree it produces
  ✓  Transform any CFG into CNF using the 5-step pipeline
  ✓  Fill a CYK table by hand for a short input
  ✓  Explain how grammar-constrained decoding restricts token sampling
  ✓  Sketch a recursive descent parser in plain JavaScript
  ✓  Sketch a grammar generator using random rule expansion
  ✓  State the time complexity of CYK and explain where O(n³) comes from
```

---

### Quick Reference Card

```
  CONCEPT          ONE-LINE SUMMARY
  ──────────────────────────────────────────────────────────────────
  Grammar          N, T, P, S — rules for rewriting non-terminals
  Derivation       Sequence of rewrite steps from S to a string
  Parse tree       Structural proof that a string matches the grammar
  Ambiguity        One string → two or more distinct parse trees
  CNF              A → B C  or  A → a  (two allowed shapes only)
  CYK              O(n³) table-filling recognizer for CNF grammars
  Constrained dec  Mask forbidden tokens at each decode step
  Rec. descent     One function per non-terminal; call stack = parse state
```
