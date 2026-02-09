---
layout: post
title: "Building a Zero-Dependency Codebase Analyzer in Node.js"
date: 2026-02-08
tags: [javascript, nodejs, cli, devtools]
description: "How I built codemap — a CLI tool that generates structural overviews of codebases using regex-based parsing, with zero runtime dependencies."
---

When you join a new project or open-source repo, the first question is always: *what's in here?* The `tree` command shows file structure, but it doesn't tell you what language each file is, what functions are defined, or how files depend on each other.

I built [codemap](https://github.com/Taru0208/codemap) — a CLI tool that generates structural overviews of codebases with zero runtime dependencies. Here's what I learned about parsing code without a parser.

## The Problem with `tree`

```
src/
├── analyzer.js
├── cli.js
├── formatter.js
└── index.js
```

This tells you nothing. What does `analyzer.js` export? How many lines is it? Does `cli.js` import from `formatter.js`? You'd have to open each file to find out.

## What codemap does differently

```
src/
├─ analyzer.js (JavaScript, 335L)
     function parseGitignore:38
     function extractSignatures:66
     function extractImports:185
├─ cli.js (JavaScript, 76L)
├─ formatter.js (JavaScript, 252L)
   ⬡ function formatSummary:26
   ⬡ function formatTree:41
   ⬡ function formatGraph:192
└─ index.js (JavaScript, 3L)
```

Language detection, line counts, function signatures with line numbers, exported symbols marked with ⬡. All without installing a single dependency.

## Design Decisions

### Why zero dependencies?

1. **Install speed** — `npx codemap-cli` runs in seconds, not minutes
2. **Trust** — No supply chain risk. You can read the entire source in 10 minutes
3. **Maintenance** — No breaking updates from upstream packages
4. **Portability** — Works on any Node.js 18+ environment

The entire tool is ~750 lines across 4 files. Every line is mine.

### Language detection without a parser

You don't need a full AST parser to extract useful information. A regex-based approach works surprisingly well for signatures:

```javascript
// JavaScript/TypeScript functions
const exportMatch = line.match(
  /^export\s+(default\s+)?(function|class|const)\s+(\w+)/
);
const funcMatch = line.match(/^(async\s+)?function\s+(\w+)/);

// Python
const defMatch = line.match(/^(async\s+)?def\s+(\w+)/);

// Go
const funcMatch = line.match(/^func\s+(?:\(.*?\)\s+)?(\w+)/);

// Rust
const fnMatch = line.match(/^(?:pub\s+)?(?:async\s+)?fn\s+(\w+)/);
```

This catches top-level declarations across 9 languages. It won't find nested functions or complex patterns, but for a structural overview, that's exactly right. You want the shape of the code, not every detail.

### Import extraction for dependency graphs

The same regex approach works for imports:

```javascript
// JS/TS: import ... from '...' or require('...')
line.match(/(?:import\s+.*\s+from\s+|import\s+)['"]([^'"]+)['"]/);
line.match(/require\s*\(\s*['"]([^'"]+)['"]\s*\)/);

// Python: from X import Y
line.match(/^from\s+([\w.]+)\s+import/);
```

With imports extracted, you can build a dependency graph and output it as Mermaid:

```
graph LR
  src_cli_js["src/cli.js"]
  src_analyzer_js["src/analyzer.js"]
  src_formatter_js["src/formatter.js"]

  src_cli_js --> src_analyzer_js
  src_cli_js --> src_formatter_js
```

This renders directly on GitHub — no external tools needed.

### Respecting .gitignore

One subtle but important feature: codemap reads `.gitignore` and applies those patterns. This means `node_modules`, `dist`, `build`, and other generated directories are automatically excluded. The output shows your *source* code, not your build artifacts.

## Output Modes

The tool outputs in four formats:

- **Tree** (default) — colored terminal output with signatures
- **Summary** (`-s`) — quick stats: file count, lines, language breakdown
- **Markdown** (`-m`) — structured tables for documentation
- **Graph** (`-g`) — Mermaid dependency diagram

The JSON mode (`-j`) outputs everything as structured data, so you can pipe to `jq` or feed into other tools:

```bash
# Find files with the most functions
codemap -j | jq '.files[] | select(.signatures | length > 5) | {name, count: (.signatures | length)}'
```

## What I'd do differently

**More language support for imports.** Currently import extraction covers JS/TS, Python, Go, Rust, and C/C++. Java and Ruby are missing. Adding them is straightforward — each language has a distinctive import syntax.

**Better resolution of relative imports.** The dependency graph resolves `./foo` to `./foo.js` or `./foo.ts`, but doesn't handle path aliases (`@/components/...`) or barrel imports (`./index`). These are solvable but add complexity.

**Performance on large repos.** The tool reads every file sequentially. For a monorepo with 10,000 files, parallel file reading with `Promise.all` batches would help. For now, it's fast enough for most projects.

## Try it

```bash
npx codemap-cli
```

Or check out the source: [github.com/Taru0208/codemap](https://github.com/Taru0208/codemap)

There's also a [GitHub Action](https://github.com/Taru0208/codemap-action) that auto-generates `CODEMAP.md` on every push.
