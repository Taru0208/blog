---
layout: post
title: "Building a GitHub Action Without @actions/core"
date: 2026-02-08
tags: [github, actions, javascript, devops]
description: "How to build a fully functional GitHub Action with zero dependencies — replacing @actions/core with environment variables and the GITHUB_OUTPUT protocol."
---

Most GitHub Action tutorials start with `npm install @actions/core @actions/github`. But you don't actually need those packages. Here's how to build a fully functional Action with zero dependencies — just Node.js.

## What we're building

A GitHub Action that runs [codemap](https://github.com/Taru0208/codemap) on your repository and auto-commits a `CODEMAP.md` file. Every push to main keeps your codebase documentation up-to-date.

## The action.yml

Every Action needs an `action.yml` that defines inputs, outputs, and how to run:

```yaml
name: 'Codemap'
description: 'Generate a structural overview of your codebase as CODEMAP.md'
branding:
  icon: 'map'
  color: 'blue'

inputs:
  path:
    description: 'Directory to analyze'
    default: '.'
  output:
    description: 'Output file path'
    default: 'CODEMAP.md'
  commit:
    description: 'Auto-commit the generated file'
    default: 'true'

runs:
  using: 'node20'
  main: 'index.js'
```

That's it. No `package.json` needed for the Action itself — GitHub runs `index.js` directly with Node 20.

## Replacing @actions/core

The `@actions/core` package provides three things: reading inputs, setting outputs, and reporting failures. All three use environment variables under the hood:

```javascript
// Reading inputs — just read INPUT_* env vars
const getInput = (name) =>
  process.env[`INPUT_${name.toUpperCase().replace(/-/g, '_')}`] || '';

// Setting outputs — append to GITHUB_OUTPUT file
const setOutput = (name, value) => {
  const filePath = process.env.GITHUB_OUTPUT;
  if (filePath) {
    const delimiter = `ghadelimiter_${Date.now()}`;
    writeFileSync(filePath, `${name}<<${delimiter}\n${value}\n${delimiter}\n`, { flag: 'a' });
  }
};

// Reporting failures — print error annotation and exit
const setFailed = (msg) => {
  console.error(`::error::${msg}`);
  process.exit(1);
};
```

That's 15 lines replacing a 200KB dependency tree.

### How GitHub passes inputs

When your workflow has:

```yaml
- uses: your/action@v1
  with:
    path: src
    output: docs/CODEMAP.md
```

GitHub sets environment variables:
- `INPUT_PATH=src`
- `INPUT_OUTPUT=docs/CODEMAP.md`

The naming convention: `INPUT_` + uppercase name with hyphens converted to underscores. So `max-depth` becomes `INPUT_MAX_DEPTH`.

### The GITHUB_OUTPUT protocol

Before November 2022, Actions used `::set-output` commands printed to stdout. Now outputs use a file-based protocol:

1. GitHub creates a temp file and sets `GITHUB_OUTPUT` to its path
2. Your Action appends to this file in a specific format
3. GitHub reads the file after your Action completes

The format uses heredoc-style delimiters:

```
output_name<<DELIMITER
value here
(can be multiline)
DELIMITER
```

## The main script

Here's the complete `index.js`:

```javascript
import { execSync } from 'child_process';
import { writeFileSync, readFileSync, existsSync } from 'fs';
import { resolve } from 'path';

const getInput = (name) =>
  process.env[`INPUT_${name.toUpperCase().replace(/-/g, '_')}`] || '';
const setOutput = (name, value) => { /* ... as above ... */ };
const setFailed = (msg) => { console.error(`::error::${msg}`); process.exit(1); };

try {
  const targetPath = getInput('path') || '.';
  const outputFile = getInput('output') || 'CODEMAP.md';
  const autoCommit = getInput('commit') !== 'false';

  // Install the tool
  execSync('npm install -g github:Taru0208/codemap', { stdio: 'inherit' });

  // Run analysis
  const output = execSync(`codemap ${targetPath} -m`, {
    encoding: 'utf8',
    maxBuffer: 10 * 1024 * 1024,
  });

  // Write result
  const existing = existsSync(outputFile)
    ? readFileSync(outputFile, 'utf8')
    : null;
  writeFileSync(resolve(outputFile), output);

  // Auto-commit if changed
  if (autoCommit && output !== existing) {
    execSync('git config user.name "github-actions[bot]"');
    execSync('git config user.email "github-actions[bot]@users.noreply.github.com"');
    execSync(`git add "${outputFile}"`);

    const status = execSync('git status --porcelain', { encoding: 'utf8' });
    if (status.trim()) {
      execSync('git commit -m "docs: update CODEMAP.md"');
      execSync('git push');
    }
  }
} catch (err) {
  setFailed(err.message);
}
```

## The workflow

Users add this to `.github/workflows/codemap.yml`:

```yaml
name: Update CODEMAP.md
on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  codemap:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - uses: Taru0208/codemap-action@v1
```

Two things to note:
1. `permissions: contents: write` is required — the Action needs to push commits
2. `actions/setup-node` is needed because we install an npm package

## Gotcha: the auto-commit loop

If your Action commits to the same branch that triggers it, you get an infinite loop: push → Action runs → commits → push → Action runs → ...

GitHub prevents this by default: commits made with the `GITHUB_TOKEN` don't trigger workflows. But if you use a PAT or install a GitHub App, they will. Stick with the default token unless you have a reason not to.

## Versioning with tags

Users reference Actions by tag: `uses: Taru0208/codemap-action@v1`. The convention:

```bash
# Create annotated tag
git tag -a v1 -m "v1.0.0"
git push origin v1

# Update v1 to point to latest commit (for patches)
git tag -d v1
git tag -a v1 -m "v1 - updated"
git push origin :refs/tags/v1
git push origin v1
```

The `v1` tag is a moving target — it always points to the latest v1.x release. Users who want stability use `v1.0.0`; users who want auto-updates use `v1`.

## The result

Every push to main, this workflow runs and generates a `CODEMAP.md` with:
- File tree with language detection
- Function signatures with line numbers
- Export markers
- Language distribution table

All without a single runtime dependency in the Action itself.

Check it out: [codemap-action](https://github.com/Taru0208/codemap-action)
