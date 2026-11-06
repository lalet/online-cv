---
layout: post
title: "How to Use Your Native Language in a Domain Path"
date: 2026-11-06
tags: [web, unicode]
---

**TLDR:** [lalet.me/ഓർമ്മ](https://lalet.me/orma/), Malayalam, right in the address bar, redirects to [the room I built there](/2026/10/16/orma-a-room-in-koovappally.html). Getting that working meant finding out Malayalam Unicode lets you spell the same word two genuinely different ways in bytes, and a plain string match on the URL only catches one of them.

## The address bar lies a little

What you see in a browser's address bar is the pretty, readable version. What actually gets sent is ASCII only, so the browser turns ഓർമ്മ into raw UTF-8 bytes and [percent-encodes](https://developer.mozilla.org/en-US/docs/Glossary/Percent-encoding) each one first. A redirect rule just needed to match that byte string and send a visitor on to `/orma/`. Easy, well documented, worked first try.

Except it didn't work for everyone.

## One letter, two valid spellings

ഓർമ്മ means memory. The letter ർ in the middle is a *chillu*, a pure consonant with no vowel attached, common at the end of a syllable in [Malayalam](https://en.wikipedia.org/wiki/Malayalam_script). Unicode allows writing it two different ways that render as the exact same glyph:

- one precomposed codepoint, U+0D7C
- or RA (U+0D30) plus a virama (U+0D4D) plus a zero-width joiner (U+200D), which most fonts turn into the identical chillu shape through a font ligature

Same letter on screen. Different bytes underneath. Which one you get depends on where the text came from: a phone's Malayalam keyboard, a desktop input method, or something copy-pasted from an old document. I'd only tested the spelling my own keyboard happened to produce.

## Finding the gap with curl

```bash
curl -I --path-as-is "https://lalet.me/%E0%B4%93%E0%B5%BC%E0%B4%AE%E0%B5%8D%E0%B4%AE"
# 301, precomposed chillu, my own spelling

curl -I --path-as-is "https://lalet.me/%E0%B4%93%E0%B4%B0%E0%B5%8D%E2%80%8D%E0%B4%AE%E0%B5%8D%E0%B4%AE"
# also 301, once the second spelling got its own rule
```

Before the fix, the second one 404'd silently. Whoever typed ഓർമ്മ on a keyboard using the decomposed spelling would have gotten a dead link with no visible reason why, since both strings look identical on screen. A redirect rule matches the literal bytes in the request, not the rendered text, so both spellings needed their own line.

## Why bother with any of this

Because a URL in your own language shouldn't come with an asterisk. A word meaning memory, in the script I grew up reading, pointing at a room from my grandfather's house, felt worth getting exactly right rather than mostly right.
