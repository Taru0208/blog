---
layout: post
title: "Your Git History Is a Goldmine"
date: 2026-02-12
tags: [git, code-analysis, refactoring, developer-tools]
description: "Git log contains more than commit messages. Here's how to find hotspots, hidden coupling, and knowledge silos in your codebase."
---

Every codebase has secrets. Not passwords (hopefully), but patterns that only emerge when you look at *how* the code changed over time, not just what it looks like right now.

Your git history records every change, every author, every file touched in every commit. Most developers only use `git log` and `git blame`. But there's much more buried in there.

## Hotspots: Where Bugs Live

A hotspot is a file that changes frequently. Not all frequent changes are bad — but research shows that files with high *change frequency* and high *complexity* are disproportionately likely to contain bugs.

You can find hotspots with one command:

```bash
git log --format=format: --name-only | sort | uniq -c | sort -rn | head -20
```

This counts how many commits touched each file. The top results are your hotspots.

On Express.js (1700+ commits), the top hotspots are:
- `package.json` (876 commits) — version bumps, expected
- `History.md` (835 commits) — changelog, expected
- `lib/response.js` (128 commits) — the core response object, genuinely hot

That third one is interesting. 128 commits means this file changes roughly once per 13 commits. If it also has high complexity, it's a prime candidate for refactoring.

**Churn** adds another dimension: total lines added + deleted. A file with 50 commits but only 200 lines of churn is probably getting small fixes. A file with 50 commits and 5000 lines of churn is being rewritten repeatedly — that's a red flag.

## Temporal Coupling: Hidden Dependencies

Here's the subtle one. Two files are *temporally coupled* if they change together in the same commit. This reveals dependencies that no import statement shows.

Classic example: you change `lib/router.js` and almost always also change `test/router.test.js`. That's expected — test follows implementation.

But what about this (real data from Express.js)?

```
621×  History.md  ↔  package.json     (92% coupling)
 57×  .travis.yml ↔  appveyor.yml     (93% coupling)
```

The first pair makes sense — every release touches both. The second is interesting: Travis CI and AppVeyor configs always change together. This suggests they should either be unified or managed by a single CI system.

Coupling becomes a problem when it's *unexpected*. If `user-service.js` and `billing-service.js` change together 80% of the time, they might not be as independent as your architecture diagram claims.

To calculate coupling degree:

```
degree = co_changes / min(changes_A, changes_B)
```

A degree of 1.0 means every time A changes, B changes too (or vice versa). Above 0.7 is worth investigating.

## Code Age: Fresh vs. Ancient

```bash
git log --format="%ai %s" --diff-filter=M -- "path/to/file" | tail -1
```

Group your files by when they were last modified:
- **Fresh** (≤7 days): actively being worked on
- **Recent** (≤30 days): current development
- **Aging** (≤90 days): might need attention
- **Stale** (≤1 year): stable or forgotten
- **Ancient** (>1 year): deep infrastructure or dead code

A healthy codebase has a mix. If 90% of files are "ancient," either the code is very stable or the project is abandoned. If 90% are "fresh," things are changing fast — make sure tests are keeping up.

Express.js: 271 ancient files, 4 fresh. Most of the framework hasn't been touched in over a year. That's stability, not neglect — it's a mature project.

## Knowledge Silos: The Bus Factor

A knowledge silo is a file that only one person has ever modified. If that person leaves, no one understands the code.

```bash
git log --format="%an" -- "path/to/file" | sort -u | wc -l
```

If that returns 1, you have a silo.

Express.js has very few silos (only `lib/middleware/query.js` with 8 commits by one author). That's good for a major open-source project — most files have multiple contributors.

But in company codebases, silos are everywhere. The "that one person who knows the billing system" problem is real, and git history makes it quantifiable.

## Making It Practical

These analyses are most useful when:

1. **Planning refactoring**: Start with hotspots. High churn + high coupling = highest impact refactoring target.
2. **Code review**: If a PR touches a file that's coupled to files not in the PR, that's a flag.
3. **Onboarding**: Show new team members the coupling graph. It reveals the *real* architecture faster than any diagram.
4. **Risk assessment**: Knowledge silos + hotspots = the riskiest parts of your codebase.

The insight is simple: static analysis tells you what your code *is*. Git history tells you what your code *does* — how it lives, breathes, and evolves. Both perspectives together give you the full picture.

## Further Reading

- Adam Tornhill, *Your Code as a Crime Scene* — the book that started this field
- [code-maat](https://github.com/adamtornhill/code-maat) — Tornhill's original analysis tool (Clojure)
- Michael Feathers' work on code age and hotspot analysis
