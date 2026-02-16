---
layout: post
title: "Build a QR Code Generator from Scratch in JavaScript"
date: 2026-02-16
tags: [javascript, qr-code, algorithm, from-scratch]
description: "A pure JavaScript QR code generator that runs anywhere — no dependencies, no canvas, just math. Outputs SVG, ASCII, or a binary matrix."
---

QR code libraries are everywhere. But have you ever wondered what happens inside one? The algorithm is surprisingly elegant — a mix of error correction math, clever bit placement, and visual pattern rules.

I built a QR code generator from scratch in pure JavaScript. No dependencies, no canvas, no node-only APIs. It runs in browsers, Cloudflare Workers, Node.js, Deno — anywhere JavaScript runs. Here's how it works.

## What a QR code actually is

A QR code is a 2D grid of black and white modules. Every QR code has three fixed elements:

1. **Finder patterns** — the three big squares in the corners (top-left, top-right, bottom-left). These let scanners detect orientation.
2. **Timing patterns** — alternating black/white lines connecting the finders. They help scanners determine module positions.
3. **Data area** — everything else. Your actual data, encoded and error-corrected.

The grid size depends on the **version** (1-40). Version 1 is 21×21 modules. Each version adds 4 modules per side: `size = 17 + (version × 4)`.

## Step 1: Encode the data

QR codes support multiple encoding modes. The simplest is **byte mode** — just UTF-8 bytes. The data stream is:

```
[Mode indicator: 4 bits] [Character count: 8 bits] [Data bytes] [Terminator] [Padding]
```

```javascript
function encodeData(text, version) {
  const data = new TextEncoder().encode(text);
  const bits = [];

  // Mode indicator: byte mode = 0100
  pushBits(bits, 0b0100, 4);

  // Character count (8 bits for versions 1-9)
  pushBits(bits, data.length, 8);

  // Data bytes
  for (const b of data) {
    pushBits(bits, b, 8);
  }

  // Terminator: up to 4 zero bits
  const capacity = totalDataBits(version);
  pushBits(bits, 0, Math.min(4, capacity - bits.length));

  // Pad to byte boundary, then alternate 0xEC/0x11
  while (bits.length % 8 !== 0) bits.push(0);
  const padBytes = [0xEC, 0x11];
  let i = 0;
  while (bits.length < capacity) {
    pushBits(bits, padBytes[i++ % 2], 8);
  }

  return bitsToBytes(bits);
}
```

## Step 2: Reed-Solomon error correction

This is the math-heavy part. QR codes use Reed-Solomon codes over GF(256) — the same error correction used in CDs and deep-space communication.

The idea: take your data bytes and compute extra "parity" bytes that can reconstruct the data if some modules are damaged.

First, you need Galois Field arithmetic:

```javascript
// GF(256) with primitive polynomial x^8 + x^4 + x^3 + x^2 + 1
const GF_EXP = new Uint8Array(512);
const GF_LOG = new Uint8Array(256);

let x = 1;
for (let i = 0; i < 255; i++) {
  GF_EXP[i] = x;
  GF_LOG[x] = i;
  x = x << 1;
  if (x & 0x100) x ^= 0x11D; // Reduction
}
```

Then polynomial division gives you the error correction codewords:

```javascript
function rsEncode(data, nsym) {
  const gen = generatorPolynomial(nsym);
  const res = [...data, ...new Array(nsym).fill(0)];

  for (let i = 0; i < data.length; i++) {
    const coef = res[i];
    if (coef !== 0) {
      for (let j = 1; j < gen.length; j++) {
        res[i + j] ^= gfMul(gen[j], coef);
      }
    }
  }
  return res.slice(data.length); // The remainder = EC codewords
}
```

The number of error correction codewords depends on the version and **error correction level** (L/M/Q/H). Level H can recover ~30% data loss, but needs more space, so your text capacity drops.

## Step 3: Build the matrix

This is where it gets visual. The matrix starts empty, then you layer on patterns:

```javascript
function buildMatrix(version, dataBits) {
  const size = 17 + version * 4;
  const matrix = Array.from({ length: size },
    () => new Uint8Array(size));

  addFinderPatterns(matrix, size);     // Three corners
  addTimingPatterns(matrix, size);     // Connecting lines
  addAlignmentPatterns(matrix, version); // Version 2+
  addDarkModule(matrix, version);      // One mandatory dark pixel

  placeData(matrix, size, dataBits);   // Zigzag data placement

  return matrix;
}
```

The data placement follows a specific zigzag pattern — two columns at a time, moving upward then downward, right to left. It skips over function patterns:

```javascript
function placeData(matrix, size, bits) {
  let bitIdx = 0;
  let upward = true;

  for (let col = size - 1; col >= 0; col -= 2) {
    if (col === 6) col = 5; // Skip timing column

    const rows = upward
      ? range(size - 1, 0, -1)
      : range(0, size - 1, 1);

    for (const row of rows) {
      for (const c of [col, col - 1]) {
        if (matrix[row][c] !== 0) continue; // Skip function patterns
        matrix[row][c] = bits[bitIdx++] ? BLACK : WHITE;
      }
    }
    upward = !upward;
  }
}
```

## Step 4: Masking

A raw QR code might have large uniform areas that confuse scanners. The spec defines 8 mask patterns — XOR operations that flip data modules based on their position:

```javascript
const MASKS = [
  (r, c) => (r + c) % 2 === 0,
  (r, c) => r % 2 === 0,
  (r, c) => c % 3 === 0,
  (r, c) => (r + c) % 3 === 0,
  // ... 4 more
];
```

You try all 8 masks and pick the one with the lowest **penalty score** — a scoring system that penalizes adjacent same-color runs, 2×2 blocks, finder-like patterns, and unbalanced dark/light ratios.

## Step 5: Output as SVG

For the output, SVG is ideal — it's text-based, resolution-independent, and works everywhere:

```javascript
function toSVG(matrix, moduleSize = 10, margin = 4) {
  const size = matrix.length;
  const total = (size + margin * 2) * moduleSize;
  let path = '';

  for (let r = 0; r < size; r++) {
    for (let c = 0; c < size; c++) {
      if (isBlack(matrix[r][c])) {
        const x = (c + margin) * moduleSize;
        const y = (r + margin) * moduleSize;
        path += `M${x},${y}h${moduleSize}v${moduleSize}h-${moduleSize}z`;
      }
    }
  }

  return `<svg xmlns="http://www.w3.org/2000/svg"
    viewBox="0 0 ${total} ${total}">
    <rect width="${total}" height="${total}" fill="#fff"/>
    <path d="${path}" fill="#000"/>
  </svg>`;
}
```

Using a single `<path>` element instead of individual `<rect>`s keeps the SVG compact — typically 40-60% smaller.

## The result

A complete QR code generator in ~400 lines of JavaScript. No dependencies. Supports versions 1-10 (up to ~270 characters with EC level L), all four error correction levels, custom colors, and three output formats:

- **SVG** — for web embedding and printing
- **ASCII** — for terminal output
- **Binary matrix** — for custom rendering

The full implementation handles edge cases the spec requires: proper block interleaving for multi-block versions, format information bits with BCH error correction, and the mysterious "dark module" at position `(4v + 9, 8)`.

## What I learned

1. **Reed-Solomon is beautiful** — the same math that corrects scratched CDs also fixes damaged QR codes. GF(256) arithmetic is surprisingly intuitive once you see it as polynomial operations.

2. **The mask selection matters** — without masking, many QR codes would be unscannable. The penalty scoring system is a clever heuristic that works well in practice.

3. **QR codes are overengineered in the right way** — the spec has redundancy at every level (finder patterns, timing patterns, format info is stored twice, data is error-corrected). This is why your phone can read a QR code even when it's tilted, partially covered, or printed on a coffee cup.

4. **Pure JS is enough** — you don't need `canvas`, `sharp`, or any native module. The entire algorithm is arithmetic and array operations. This makes it deployable on Cloudflare Workers, Deno, browsers, or any JS runtime.
