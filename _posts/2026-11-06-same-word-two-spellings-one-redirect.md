---
layout: post
title: "How to Use Your Native Language in a Domain Path"
date: 2026-11-06
tags: [web, unicode]
---

<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+Malayalam:wght@400;500;700&display=swap" rel="stylesheet">

<div class="lang-toggle" role="group" aria-label="Language">
  <button type="button" class="lang-btn active" data-lang="en">English</button>
  <button type="button" class="lang-btn" data-lang="ml">മലയാളം</button>
</div>

<style>
.lang-toggle { display: inline-flex; gap: 4px; background: #f4f4f4; border-radius: 999px; padding: 4px; margin-bottom: 1.4em; }
.lang-btn { border: none; background: transparent; padding: 6px 18px; border-radius: 999px; font-size: 0.9em; cursor: pointer; color: #556; }
.lang-btn.active { background: #fff; box-shadow: 0 1px 2px rgba(0,0,0,0.08); font-weight: 600; color: #222; }
#lang-ml { font-family: 'Noto Sans Malayalam', sans-serif; }
.font-credit { font-size: 0.8em; color: #97AAC3; margin-top: 2em; border-top: 1px solid #eee; padding-top: 1em; }
</style>

<script>
document.querySelectorAll('.lang-btn').forEach(function (btn) {
  btn.addEventListener('click', function () {
    document.querySelectorAll('.lang-btn').forEach(function (b) { b.classList.remove('active'); });
    btn.classList.add('active');
    var lang = btn.dataset.lang;
    document.getElementById('lang-en').style.display = lang === 'en' ? '' : 'none';
    document.getElementById('lang-ml').style.display = lang === 'ml' ? '' : 'none';
  });
});
</script>

<div id="lang-en" markdown="1">

**TLDR:** [lalet.me/ഓർമ്മ](https://lalet.me/orma/), Malayalam, right in the address bar, redirects to [the room I built there](https://lalet.me/orma/). Getting that working meant finding out Malayalam Unicode lets you spell the same word two genuinely different ways in bytes, and a plain string match on the URL only catches one of them.

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

<p class="font-credit">Set in <a href="https://fonts.google.com/noto/specimen/Noto+Sans+Malayalam">Noto Sans Malayalam</a> (Google, SIL Open Font License). Thanks to <a href="https://smc.org.in/">Swathanthra Malayalam Computing</a> and the wider FOSS Malayalam community, whose fonts, keyboards and typing tools made it possible to even have this conversation in script, let alone get it exactly right.</p>

</div>

<div id="lang-ml" markdown="1" style="display: none;">

**ചുരുക്കം:** [lalet.me/ഓർമ്മ](https://lalet.me/orma/) എന്ന് മലയാളത്തിൽ തന്നെ അഡ്രസ് ബാറിൽ ടൈപ്പ് ചെയ്താൽ, [ഞാൻ ഉണ്ടാക്കിയ ആ മുറിയിലേക്ക്](/2026/10/16/orma-a-room-in-koovappally.html) എത്തും. ഇത് ശരിയാക്കാൻ നോക്കിയപ്പോഴാണ് മനസ്സിലായത്, മലയാളം യൂണികോഡിൽ ഒരേ വാക്ക് ബൈറ്റ് ലെവലിൽ രണ്ട് വിധത്തിൽ എഴുതാമെന്ന്. ഒരു സാധാരണ സ്ട്രിംഗ് മാച്ചിന് അതിലൊന്നേ പിടികിട്ടൂ.

## അഡ്രസ് ബാർ അൽപ്പം നുണ പറയും

ബ്രൗസറിന്റെ അഡ്രസ് ബാറിൽ കാണുന്നത് മനോഹരമായ, വായിക്കാൻ പറ്റുന്ന രൂപമാണ്. പക്ഷേ യഥാർത്ഥത്തിൽ പോകുന്നത് ASCII മാത്രം. അതുകൊണ്ട് ബ്രൗസർ ഓർമ്മ എന്ന വാക്കിനെ UTF-8 ബൈറ്റുകളാക്കി മാറ്റി, ഓരോന്നും [percent-encode](https://developer.mozilla.org/en-US/docs/Glossary/Percent-encoding) ചെയ്യും. പിന്നെ ഒരു റീഡയറക്ട് റൂൾ ആ ബൈറ്റ് സ്ട്രിംഗ് മാച്ച് ചെയ്ത് `/orma/` ലേക്ക് അയച്ചാൽ മതി. എളുപ്പം തോന്നി, നന്നായി ഡോക്യുമെന്റ് ചെയ്തതാണ്, ആദ്യ ട്രൈയിൽ തന്നെ വർക്ക് ചെയ്തു.

പക്ഷേ എല്ലാവർക്കും വേണ്ടി വർക്ക് ചെയ്തില്ല.

## ഒരു അക്ഷരം, ശരിയായ രണ്ട് എഴുത്തുരീതികൾ

ഓർമ്മ എന്ന് പറഞ്ഞാൽ memory എന്നാണ്. നടുക്കുള്ള ർ ഒരു *ചില്ല്* ആണ്, സ്വരം ചേരാത്ത വെറും വ്യഞ്ജനം, [മലയാളത്തിൽ](https://en.wikipedia.org/wiki/Malayalam_script) വാക്കിന്റെ അവസാനം സാധാരണമായി വരുന്നത്. യൂണികോഡിൽ ഇത് രണ്ട് വിധത്തിൽ എഴുതാം, രണ്ടും സ്ക്രീനിൽ ഒരേ പോലെ കാണും:

- ഒരു precomposed codepoint, U+0D7C
- അല്ലെങ്കിൽ RA (U+0D30) പിന്നെ ഒരു വിരാമം (U+0D4D) പിന്നെ ഒരു zero-width joiner (U+200D). മിക്ക ഫോണ്ടുകളും ഇതിനെ ഒരു ligature വെച്ച് അതേ ചില്ല് രൂപമാക്കി കാണിക്കും

സ്ക്രീനിൽ ഒരേ അക്ഷരം, അടിയിൽ വേറെ ബൈറ്റുകൾ. ഏതാണ് കിട്ടുക എന്നത് ടെക്സ്റ്റ് എവിടുന്ന് വന്നു എന്നതിനെ ആശ്രയിച്ചിരിക്കും: ഫോണിലെ മലയാളം കീബോർഡ്, ഡെസ്ക്ടോപ്പ് ഇൻപുട്ട് മെത്തേഡ്, അല്ലെങ്കിൽ പഴയ ഏതോ ഡോക്യുമെന്റിൽ നിന്ന് കോപ്പി-പേസ്റ്റ് ചെയ്തത്. ഞാൻ ടെസ്റ്റ് ചെയ്തത് എന്റെ സ്വന്തം കീബോർഡ് ഉണ്ടാക്കുന്ന സ്പെല്ലിംഗ് മാത്രമായിരുന്നു.

## curl വെച്ച് പ്രശ്നം കണ്ടുപിടിച്ചത്

```bash
curl -I --path-as-is "https://lalet.me/%E0%B4%93%E0%B5%BC%E0%B4%AE%E0%B5%8D%E0%B4%AE"
# 301, precomposed ചില്ല്, എന്റെ സ്വന്തം സ്പെല്ലിംഗ്

curl -I --path-as-is "https://lalet.me/%E0%B4%93%E0%B4%B0%E0%B5%8D%E2%80%8D%E0%B4%AE%E0%B5%8D%E0%B4%AE"
# ഇതും 301, രണ്ടാമത്തെ സ്പെല്ലിംഗിനും സ്വന്തം റൂൾ കിട്ടിയപ്പോൾ
```

ഫിക്സ് ചെയ്യുന്നതിന് മുമ്പ് രണ്ടാമത്തേത് നിശബ്ദമായി 404 ആയിരുന്നു. decomposed സ്പെല്ലിംഗ് വെച്ച് ഓർമ്മ ടൈപ്പ് ചെയ്ത ആർക്കും, രണ്ട് സ്ട്രിംഗും സ്ക്രീനിൽ ഒരേ പോലെ കാണുന്നത് കൊണ്ട്, കാരണം പോലും അറിയാതെ ഒരു ഡെഡ് ലിങ്ക് കിട്ടുമായിരുന്നു. റീഡയറക്ട് റൂൾ നോക്കുന്നത് റിക്വസ്റ്റിലെ യഥാർത്ഥ ബൈറ്റുകളാണ്, കാണുന്ന ടെക്സ്റ്റല്ല. അതുകൊണ്ട് രണ്ട് സ്പെല്ലിംഗിനും സ്വന്തം വരി വേണ്ടി വന്നു.

## എന്തിനാ ഇത്രയും കഷ്ടപ്പെടുന്നത്

കാരണം, സ്വന്തം ഭാഷയിലുള്ള ഒരു URL-ൽ ഒരു നക്ഷത്രചിഹ്നം (asterisk) വേണ്ട എന്ന് തോന്നി. ഓർമ്മ എന്നർത്ഥമുള്ള ഒരു വാക്ക്, ഞാൻ വളർന്ന ലിപിയിൽ, അപ്പൂപ്പന്റെ വീട്ടിലെ ഒരു മുറിയിലേക്ക് ചൂണ്ടിക്കാണിക്കുമ്പോൾ, ഏകദേശം ശരിയാക്കിയാൽ പോരാ, കൃത്യമായി തന്നെ വേണം എന്ന് തോന്നി.

<p class="font-credit">Set in <a href="https://fonts.google.com/noto/specimen/Noto+Sans+Malayalam">Noto Sans Malayalam</a> (Google, SIL Open Font License). Thanks to <a href="https://smc.org.in/">Swathanthra Malayalam Computing</a> and the wider FOSS Malayalam community, whose fonts, keyboards and typing tools made it possible to even have this conversation in script, let alone get it exactly right.</p>

</div>
