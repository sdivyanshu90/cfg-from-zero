# CFG From Zero

An offline, self-contained interactive course that takes you from the definition of a context-free grammar all the way to building a grammar-constrained structured-output parser — with visualizations at every step.

---

## Quick Start

The app fetches lesson markdown files at runtime, so it must be served over HTTP (not opened as a local file).

```bash
# from the project root
python3 -m http.server 8000
```

Then open **http://127.0.0.1:8000** in your browser.

No build step, no dependencies, no internet connection required.

---

## What You Will Learn

| # | Chapter | Key Ideas |
|---|---------|-----------|
| 1 | **What Is a Grammar?** | Chomsky hierarchy, CFG 4-tuple (N, T, P, S), terminals vs non-terminals, operator precedence encoding |
| 2 | **Derivations and Parse Trees** | Rewrite steps, leftmost/rightmost derivation, parse tree construction, ambiguity |
| 3 | **Chomsky Normal Form** | 5-step normalization pipeline — epsilon removal, unit removal, useless symbols, binarization, terminal isolation |
| 4 | **CYK Parsing Algorithm** | Bottom-up chart parsing, triangular DP table, O(n³) complexity, parse recovery via back-pointers |
| 5 | **Grammar-Constrained Generation** | Token masking, prefix filtering, incremental parse state, Outlines / Guidance / GBNF |
| 6 | **Build Your Own Mini Parser** | Recursive descent implementation, CYK builder lab, AST evaluation, random string generation |

---

## Project Structure

```
cfg-from-zero/
├── index.html          # Single-file app — all UI, styles, and logic
└── lessons/
    ├── chapter-1-what-is-a-grammar.md
    ├── chapter-2-derivations-and-parse-trees.md
    ├── chapter-3-chomsky-normal-form.md
    ├── chapter-4-cyk-parsing-algorithm.md
    ├── chapter-5-structured-output-parsing.md
    └── chapter-6-build-your-own-mini-parser.md
```

`index.html` embeds all JavaScript and CSS. Each lesson markdown file is fetched by the app when its chapter is opened and rendered inside the interactive panel.

---

## Lesson Format

Each lesson file provides:

- **Concept explanations** with formal definitions and worked examples
- **ASCII diagrams** — grammar hierarchies, parse trees, CYK tables, call stack traces
- **Annotated code snippets** — JavaScript implementations for each algorithm
- **Self-check tables** — questions with answers to test understanding before moving on

---

## Covered Algorithms

- **Grammar normalization** — full 5-step CNF transformation with before/after diffs
- **CYK parsing** — triangular DP table filled bottom-up with split-point enumeration
- **Recursive descent parsing** — one function per non-terminal, explicit call stack trace
- **Grammar-constrained decoding** — logit masking, prefix filtering, incremental recognizer state
- **Random grammar generation** — forward expansion of non-terminals to produce sample strings