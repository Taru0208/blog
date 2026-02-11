---
layout: post
title: "The Math Inside Korean Text"
date: 2026-02-12
tags: [unicode, korean, algorithms]
description: "Korean syllable blocks aren't arbitrary shapes — they're algorithmically composed. You can decompose any Korean character with three lines of arithmetic."
---

Korean has a writing system unlike any other. Hangul syllable blocks aren't arbitrary shapes — they're algorithmically composed. And Unicode encodes this structure directly, which means you can decompose any Korean character with nothing but arithmetic.

## The Structure

Every Korean syllable block is built from up to three components:

- **초성** (choseong): initial consonant — 19 possible
- **중성** (jungseong): medial vowel — 21 possible
- **종성** (jongseong): final consonant — 28 possible (including "none")

The character `한` decomposes into: `ㅎ` + `ㅏ` + `ㄴ`. The character `가` is just `ㄱ` + `ㅏ` (no final consonant).

This gives us 19 × 21 × 28 = **11,172** possible syllable blocks. And Unicode assigns them all contiguous code points, starting at `U+AC00`.

## The Formula

The encoding follows a simple formula:

```
code = (cho × 21 + jung) × 28 + jong + 0xAC00
```

Which means decomposition is just modular arithmetic:

```javascript
function decompose(char) {
  const code = char.charCodeAt(0) - 0xAC00;
  const jong = code % 28;
  const jung = ((code - jong) / 28) % 21;
  const cho = Math.floor(((code - jong) / 28) / 21);
  return { cho, jung, jong };
}
```

Three lines. No lookup tables for the decomposition itself. The structure is in the numbers.

## Why This Matters

This isn't just an encoding trick. It reflects something real about the writing system.

King Sejong designed Hangul in 1443 with systematic principles. Consonants encode the mouth shape used to produce them. Vowels are built from three elements: a horizontal line (earth), a vertical line (human), and a dot (heaven, now a short stroke). The system was designed to be learnable — and it is. Korea has near-100% literacy.

The Unicode encoding preserves this systematism. When you do `code % 28` to extract the final consonant, you're not just bit-shifting — you're peeling apart a writing system that was designed to be peeled apart.

## Practical Use: Text Analysis

Once you can decompose syllables, you can analyze Korean text in ways that aren't possible by treating characters as opaque.

**Rhyme detection.** Two syllables rhyme if they share the same vowel and final consonant. `산`, `간`, `만` all end in `ㅏ` + `ㄴ`.

**Vowel harmony.** Korean has a vestigial vowel harmony system inherited from Middle Korean. "Bright" vowels (`ㅏ`, `ㅗ`) pair with bright, "dark" vowels (`ㅓ`, `ㅜ`) pair with dark. This still shows up in onomatopoeia: `반짝반짝` (sparkle, bright) vs `번쩍번쩍` (flash, dark). Same meaning, different feel.

**Syllable weight.** Syllables with a final consonant (받침) are "heavier." Poetry with many closed syllables feels denser. Open syllables flow more lightly.

I analyzed three famous Korean poems:

| Poem | Open syllables | Vowel tendency |
|------|---------------|----------------|
| 서시 (윤동주) | 59.2% | Bright (양성) |
| 진달래꽃 (김소월) | 67.4% | Bright (양성) |
| 꽃 (김춘수) | 66.1% | Bright (양성) |

All three lean bright and open — but 서시 has notably more closed syllables, giving it a heavier, more deliberate feel. 진달래꽃 is the lightest, with the highest proportion of open syllables. You can feel this when you read them aloud, even before you count.

## The Composition Direction

You can also go the other way — composing syllables from jamo:

```javascript
function compose(cho, jung, jong = 0) {
  return String.fromCharCode(0xAC00 + (cho * 21 + jung) * 28 + jong);
}
```

This lets you take a word, shift all its vowels one step darker, and produce a new (possibly nonsense) word. Or generate all possible syllables that rhyme with a given one. Or build a Korean text input method from scratch.

## What I Find Beautiful

Most writing systems are encoded as flat sequences. Latin letters, Chinese characters, Japanese kana — their Unicode positions don't encode internal structure. They're just assigned numbers.

Hangul is different. The encoding *is* the structure. The math *is* the linguistics. When you subtract `0xAC00` and start dividing, you're not reverse-engineering an encoding — you're reading the blueprint that was designed 583 years ago.

---

*The full analyzer handles decomposition, composition, rhyme detection, vowel harmony analysis, and syllable structure mapping — about 150 lines of JavaScript with zero dependencies.*
