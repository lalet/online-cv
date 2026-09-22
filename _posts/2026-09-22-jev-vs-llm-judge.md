---
layout: post
title: "Jev vs. LLM-as-Judge: Getting on the Hype Train"
date: 2026-09-22
tags: [ai, rag, llm-as-judge, evaluation]
---

Most RAG (Retrieval-Augmented Generation) systems score answers two ways: faithfulness (is it based on the retrieved text, or made up?) and relevancy (does it actually answer the question?). The standard approach is a second LLM call that returns `{"faithfulness": 1-5, "relevancy": 1-5}` as JSON. I wanted to test this against a purpose-built scoring model instead.

## What I'm comparing

[Jev](https://typesafe.ai/), from TypeSafe AI, works differently. It's a non-autoregressive model, called a "System One" model. It takes information and a set of typed questions, then returns a probability distribution over fixed answers directly. No token generation, no JSON to parse. TypeSafe trains it with [RLCD, Reinforcement Learning for Calibrated Decisions](https://typesafe.ai/blog/introducing-system-one-models-and-jev). The goal is different from typical RLHF: if the model says "90% confident," it should be right 90% of the time.

## The setup

I created ten question/context/answer pairs, each testing a different problem a real RAG judge must handle:

- A fully correct answer
- A hallucinated name and date
- A factually-true-but-off-topic answer
- A number wrong by 3x
- A question built on a false premise
- A debunked myth
- Right digits with the wrong unit
- An honest "I don't know"
- A compound answer where one part out of three is wrong
- A subjective question with no ground truth

Each one went to both judges in the same run:

- **The LLM judge**: `anthropic/claude-haiku-4.5` via OpenRouter, asked to return the JSON scores above.
- **Jev**: the same question, context, and answer as `state`, with two `score`-type questions (faithfulness, relevancy) on a 3-level scale, sent to `POST /v1/systemone`.

## What actually happened

Jev's ten calls averaged 227ms (median 158ms; the first call was slow at 839ms, likely a cold connection, the rest ran 100-270ms). The LLM judge averaged 962ms (median 874ms). Skipping token generation and JSON parsing shows up directly in speed: roughly **4-5x faster** on identical inputs.

On eight of the ten cases, both judges reached the same verdict, even when exact numbers differed. On one, they flatly disagreed.

That case: I asked "what's the capital of France?" and gave it an answer about France's population and location instead, with no mention of Paris. It's a true answer, just not one that answers the question. Jev scored it 0.02 on faithfulness, basically "not grounded at all," and was 97% sure of it. The LLM judge scored the same answer a perfect 5, "fully grounded." Not a small gap; opposite verdicts.

Think of it like a closed-book exam where the context passage is the one page you can use. Jev's question was strict: is everything in this answer actually written on that page? The page never mentions population or location, so Jev said the answer isn't grounded, full stop, even though both facts happen to be true in real life. The LLM judge asked a looser question without meaning to: is anything in this answer factually wrong? Since France's population and location are both true, it approved the answer, even though neither fact was actually on the page it received.

Same word, "faithful," two different tests. Worth knowing before you swap one judge for the other and assume a passing score means the same thing from both.

Jev also returns a confidence value per answer, based on how concentrated its probability distribution is. It's a genuinely useful signal. Across all 20 scores in this run (10 cases × 2 dimensions), the three lowest were:

- The compound-answer case's faithfulness score: **0.27** (its probabilities split almost evenly, 0.51 vs. 0.49, between "not grounded" and "partially grounded")
- The wrong-unit case's relevancy score: **0.30**
- The compound-answer case's relevancy score: **0.39**

Both of those cases are genuinely ambiguous by design: is substituting one correct-sounding color for another in a three-part answer "partially grounded" or just wrong? Does a wrong number that's still on-topic count as fully addressing the question? Jev didn't resolve that ambiguity by picking a side confidently; it flagged it. Every other case landed at 0.68 confidence or higher. A bare integer score from an LLM judge can't tell you which answers it was actually unsure about; a confidently-wrong 1/5 looks identical to a genuinely-torn 1/5.

Where the LLM judge earned its keep was the "notes" field. On the compound-answer case it wrote "contradicts the retrieved context by stating green instead of yellow." On the wrong-unit case it named the exact number that was off. Jev returns numbers and a legend, no prose, so if a human needs to understand why something was flagged, that explanation still has to come from somewhere else.

## The math, with a real example

Jev's `score` is a probability-weighted average over the level indices. For the false-premise case's relevancy question, the real response was:

```json
{ "0": 0, "1": 0.15, "2": 0.85 }
```

```
score = (0 × 0) + (1 × 0.15) + (2 × 0.85) = 1.85
```

This matches what came back (1.84, off by float rounding). `confidence` is a separate number measuring how concentrated that distribution is, not correctness. So a confident wrong answer and an honestly uncertain one are distinguishable in a way a single sampled integer never is.

## Takeaway

For a bounded scoring step like this (faithfulness and relevancy on a fixed scale), a typed-decision model is a better architectural fit than reusing a generation model as judge. Same directional judgments in almost every case, several times faster, and a genuine confidence signal for triaging what needs a closer look. The one real gap is the definitional one: "faithful" meant something different to each judge on the off-topic case, worth resolving explicitly before trusting either one blindly. My own rule from this: route anything under about 0.5 confidence to a human, or to a fallback LLM pass when the explanation matters more than the score.
