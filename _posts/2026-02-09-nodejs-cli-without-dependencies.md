---
layout: post
title: "Build a Node.js CLI Tool Without Any Dependencies"
date: 2026-02-09
tags: [nodejs, javascript, cli, tutorial]
description: "A step-by-step guide to building a production-quality CLI tool in Node.js using only built-in modules. No Commander, no Chalk, no Yargs — just Node."
---

Most CLI tool tutorials start with `npm install commander chalk`. But Node.js has everything you need built in. No dependency means no supply chain risk, no version conflicts, and instant install times.

This guide shows you how to build a real CLI tool — with argument parsing, colored output, help text, and file I/O — using nothing but Node.js built-in modules.

## Why zero dependencies?

Before we start, let's be clear about *why*:

- **Install speed**: `npm install -g your-tool` takes milliseconds instead of seconds
- **Security**: Zero attack surface from third-party code
- **Maintenance**: No Dependabot alerts, no breaking changes from upstream
- **Size**: Your package stays under 20KB instead of pulling 50+ transitive dependencies

This isn't about avoiding dependencies on principle. It's about recognizing that for many CLI tools, built-in Node.js APIs are *enough*.

## Project setup

```bash
mkdir my-cli && cd my-cli
npm init -y
```

Edit `package.json`:

```json
{
  "name": "my-cli",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "mycli": "src/cli.js"
  },
  "engines": {
    "node": ">=18"
  }
}
```

The key fields:
- `"type": "module"` — use ES modules (`import`/`export`)
- `"bin"` — tells npm what command to register globally
- `"engines"` — Node 18+ for modern API support

## 1. Argument parsing (no Yargs needed)

Commander and Yargs are great, but `process.argv` is straightforward for most tools:

```javascript
// src/cli.js
#!/usr/bin/env node

const args = process.argv.slice(2);
const flags = new Set(args.filter(a => a.startsWith('-')));
const positional = args.filter(a => !a.startsWith('-'));
```

That gives you everything. For flags with values (like `--depth 3`):

```javascript
function getFlagValue(args, flag, shortFlag) {
  const idx = args.findIndex(a => a === flag || a === shortFlag);
  if (idx === -1 || idx + 1 >= args.length) return null;
  return args[idx + 1];
}

const depth = getFlagValue(args, '--depth', '-d');
const output = getFlagValue(args, '--output', '-o');
```

This handles `mycli --depth 3 ./src` and `mycli -d 3 ./src` identically.

For boolean flags, just check the Set:

```javascript
const verbose = flags.has('--verbose') || flags.has('-v');
const help = flags.has('--help') || flags.has('-h');
```

### Help text

```javascript
if (help) {
  console.log(`mycli — do something useful

Usage: mycli [directory] [options]

Options:
  -d, --depth N    Maximum depth (default: unlimited)
  -o, --output F   Output file
  -v, --verbose    Show detailed output
  -h, --help       Show this help`);
  process.exit(0);
}
```

Plain text, no templating library needed.

## 2. Colored output (no Chalk needed)

Node.js supports ANSI escape codes natively. Here's a minimal color utility:

```javascript
// src/colors.js
const enabled = process.stdout.isTTY && !process.env.NO_COLOR;

const wrap = (code, reset) => enabled
  ? s => `\x1b[${code}m${s}\x1b[${reset}m`
  : s => s;

export const bold   = wrap(1, 22);
export const dim    = wrap(2, 22);
export const red    = wrap(31, 39);
export const green  = wrap(32, 39);
export const yellow = wrap(33, 39);
export const cyan   = wrap(36, 39);
```

Usage:

```javascript
import { bold, green, red, dim } from './colors.js';

console.log(bold('Results:'));
console.log(green('✓ 5 files processed'));
console.log(red('✗ 2 errors'));
console.log(dim('(completed in 42ms)'));
```

This automatically disables colors when:
- Output is piped (`stdout.isTTY` is false)
- `NO_COLOR` environment variable is set (respects the [no-color standard](https://no-color.org/))

That's everything Chalk does for basic use cases, in 8 lines.

## 3. File system operations

Node's `fs/promises` API is clean and modern:

```javascript
import { readdir, readFile, stat, writeFile } from 'fs/promises';
import { join, resolve, extname, basename } from 'path';

async function processDirectory(dir) {
  const entries = await readdir(dir, { withFileTypes: true });

  for (const entry of entries) {
    const fullPath = join(dir, entry.name);

    if (entry.isDirectory()) {
      await processDirectory(fullPath); // recurse
    } else if (entry.isFile()) {
      const content = await readFile(fullPath, 'utf-8');
      const stats = await stat(fullPath);
      console.log(`${entry.name}: ${stats.size} bytes, ${content.split('\n').length} lines`);
    }
  }
}
```

The `withFileTypes: true` option in `readdir` is important — it gives you `Dirent` objects that know whether each entry is a file or directory, without extra `stat()` calls.

## 4. Recursive directory walking

A reusable async directory walker:

```javascript
async function* walk(dir, ignore = []) {
  const entries = await readdir(dir, { withFileTypes: true });

  for (const entry of entries) {
    if (ignore.includes(entry.name)) continue;
    if (entry.name.startsWith('.')) continue;

    const fullPath = join(dir, entry.name);

    if (entry.isDirectory()) {
      yield* walk(fullPath, ignore);
    } else {
      yield { path: fullPath, name: entry.name, ext: extname(entry.name) };
    }
  }
}

// Usage
const ignore = ['node_modules', '.git', 'dist'];
for await (const file of walk('./src', ignore)) {
  console.log(file.path);
}
```

Async generators are perfect here — they process files one at a time without loading the entire directory tree into memory.

## 5. Progress and spinners

For long-running operations, a simple spinner:

```javascript
function spinner(message) {
  const frames = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'];
  let i = 0;

  const interval = setInterval(() => {
    process.stdout.write(`\r${frames[i++ % frames.length]} ${message}`);
  }, 80);

  return {
    stop(finalMessage) {
      clearInterval(interval);
      process.stdout.write(`\r✓ ${finalMessage}\n`);
    }
  };
}

// Usage
const s = spinner('Processing files...');
await doWork();
s.stop('Done! Processed 42 files.');
```

This writes to stdout using `\r` (carriage return) to overwrite the same line — the same technique that ora and cli-spinners use internally.

## 6. Error handling

Good CLI tools distinguish between user errors and bugs:

```javascript
function exitWithError(message, code = 1) {
  console.error(`Error: ${message}`);
  process.exit(code);
}

// Validate input before doing work
const target = positional[0];
if (!target) {
  exitWithError('No directory specified. Run with --help for usage.');
}

try {
  await stat(resolve(target));
} catch {
  exitWithError(`Directory not found: ${target}`);
}

// Wrap the main logic
try {
  await run(target, { depth, verbose });
} catch (err) {
  if (verbose) {
    console.error(err.stack);
  } else {
    exitWithError(err.message);
  }
}
```

## 7. Testing with Node's built-in test runner

Node 18+ has a built-in test runner — no need for Jest or Mocha:

```javascript
// src/cli.test.js
import { describe, it } from 'node:test';
import assert from 'node:assert';
import { execSync } from 'child_process';

describe('CLI', () => {
  it('shows help text', () => {
    const output = execSync('node src/cli.js --help').toString();
    assert.ok(output.includes('Usage:'));
  });

  it('handles missing directory gracefully', () => {
    try {
      execSync('node src/cli.js /nonexistent 2>&1');
      assert.fail('Should have thrown');
    } catch (err) {
      assert.ok(err.stdout.toString().includes('Error:'));
    }
  });

  it('processes current directory', () => {
    const output = execSync('node src/cli.js .').toString();
    assert.ok(output.length > 0);
  });
});
```

Run with:

```bash
node --test src/*.test.js
```

No config files, no plugins, no `jest.config.js`.

## Putting it all together

Here's the complete structure:

```
my-cli/
├── package.json
└── src/
    ├── cli.js        # Entry point + arg parsing
    ├── colors.js     # ANSI color helpers
    ├── core.js       # Main logic
    └── cli.test.js   # Tests
```

The CLI entry point ties everything together:

```javascript
#!/usr/bin/env node
import { resolve } from 'path';
import { stat } from 'fs/promises';
import { bold, green, red } from './colors.js';
import { analyze } from './core.js';

const args = process.argv.slice(2);
const flags = new Set(args.filter(a => a.startsWith('-')));
const positional = args.filter(a => !a.startsWith('-'));

if (flags.has('--help') || flags.has('-h')) {
  console.log(`mycli — analyze your project\n\nUsage: mycli [dir] [options]\n\n  -v, --verbose    Detailed output\n  -h, --help       Show help`);
  process.exit(0);
}

const target = resolve(positional[0] || '.');

try {
  await stat(target);
} catch {
  console.error(red(`Error: ${target} not found`));
  process.exit(1);
}

const result = await analyze(target);
console.log(bold(`${green('✓')} ${result.files} files analyzed`));
```

## Publishing to npm

When you're ready:

```bash
# Test that the bin link works
npm link
mycli --help

# Publish
npm publish
```

Users install with:

```bash
npx mycli ./my-project      # One-time use
npm install -g mycli         # Global install
```

## When *should* you use dependencies?

Not every CLI tool should be zero-dependency. Use dependencies when:

- **Complex argument parsing**: Subcommands, nested flags, validation → Commander
- **Rich terminal UI**: Tables, prompts, select menus → Inquirer, clack
- **Network requests**: HTTP clients with retry/timeout → undici (or built-in `fetch` on Node 18+)
- **Cross-platform paths**: Complex path manipulation → globby

The decision is simple: if a built-in API does the job, use it. If you'd spend hours reimplementing something complex, use a dependency.

## Further reading

- [Node.js CLI Best Practices](https://github.com/lirantal/nodejs-cli-apps-best-practices) — comprehensive guide
- [codemap](https://github.com/Taru0208/codemap) — real example of a zero-dependency CLI tool (codebase analyzer)
- [Node.js API docs](https://nodejs.org/docs/latest/api/) — `fs`, `path`, `process`, `test`

---

*Built a CLI tool with this approach? I'd like to hear about it. Find me on [GitHub](https://github.com/Taru0208).*
