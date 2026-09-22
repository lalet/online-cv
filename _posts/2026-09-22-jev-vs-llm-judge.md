---
layout: post
title: "Jev vs. LLM-as-Judge: The Real Numbers, Not the Ones I Started With"
date: 2026-09-22
tags: [ai, rag, llm-as-judge, evaluation]
---

I started with a full comparison already written for me: ten test cases, a 10x latency win, a claim that both judges agreed on every single case. It looked great. I ran it for real anyway, since I don't trust numbers I didn't measure myself enough to put on this blog. The real results are messier and, honestly, more interesting.

## What I'm comparing

Most RAG pipelines score a generated answer on two things: faithfulness (is it based on the retrieved context, or made up?) and relevancy (does it actually answer the question?). The standard way is a second LLM call, asked to return `{"faithfulness": 1-5, "relevancy": 1-5}` as JSON.

[Jev](https://typesafe.ai/), from TypeSafe AI, does this differently. It's a non-autoregressive model, what they call a "System One" model. It takes a block of information and a set of typed questions and returns a probability distribution over a fixed set of answers directly. No token generation, no JSON to parse. TypeSafe trains it with [RLCD, Reinforcement Learning for Calibrated Decisions](https://typesafe.ai/blog/introducing-system-one-models-and-jev). This is a different goal from the usual RLHF: if the model says "90% confident," it should actually be right 90% of the time.

## The setup

I wrote ten question/context/answer pairs, each testing a different failure mode a real RAG judge has to handle:

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

Each one went to both judges, for real, in the same run:

- **The LLM judge**: `anthropic/claude-haiku-4.5` via OpenRouter, asked to return the JSON scores above.
- **Jev**: the same question, context, and answer as `state`, with two `score`-type questions (faithfulness, relevancy) on a 3-level scale, sent to `POST /v1/systemone`.

## What actually happened

**Latency.** Jev's ten calls averaged 227ms (median 158ms; the first call was slow at 839ms, likely a cold connection, the rest ran 100-270ms). The LLM judge averaged 962ms (median 874ms). That's roughly **4-5x faster**, not 10x. Still a real, structural difference, just not the number I was handed.

**Direction mostly matched, with one genuine exception.** On eight of the ten cases, both judges agreed on which way an answer leaned, high or low, on both dimensions. The exception: the off-topic-but-true answer (answering "what's the capital of France?" with population and location instead). Jev scored its faithfulness at 0.02, basically "not grounded," with 0.97 confidence. The LLM judge gave it a 5, "fully grounded." That's not a difference of degree, it's the opposite verdict.

I think I know why: the LLM judge seems to read "faithful" as "not factually wrong," so a true-but-irrelevant fact passes. Jev's question was explicitly about groundedness *in the provided context*, and the context passage never mentions population or location at all. So by that stricter reading it's ungrounded regardless of whether the claim is independently true. Same word, two different tests. That's worth knowing if you're swapping one judge for the other and expecting the criteria to mean the same thing.

**The confidence signal is real, and it pointed at the right cases.** Jev returns a confidence value per answer, based on how concentrated its probability distribution is. Across all 20 scores in this run (10 cases × 2 dimensions), the three lowest were:

- The compound-answer case's faithfulness score: **0.27** (its probabilities split almost evenly, 0.51 vs. 0.49, between "not grounded" and "partially grounded")
- The wrong-unit case's relevancy score: **0.30**
- The compound-answer case's relevancy score: **0.39**

Both of those cases are genuinely ambiguous by design: is substituting one correct-sounding color for another in a three-part answer "partially grounded" or just wrong? Does a wrong number that's still on-topic count as fully addressing the question? Jev didn't resolve that ambiguity by picking a side confidently. It flagged it. Every other case landed at 0.68 confidence or higher. A bare integer score from an LLM judge can't tell you which of its answers it was actually unsure about. A confidently-wrong 1/5 looks identical to a genuinely-torn 1/5.

**The LLM judge's free-text notes are still useful.** Where the LLM judge earned its keep was in the "notes" field. On the compound-answer case it wrote "contradicts the retrieved context by stating green instead of yellow." On the wrong-unit case it named the exact number that was off. Jev returns numbers and a legend, no prose. If a human needs to understand *why* something was flagged, that context still has to come from somewhere.

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

The directional agreement and the confidence signal both held up under a real run. That part of the pitch is genuine. The specific numbers I was first given weren't: the real speedup is closer to 4-5x than 10x, and the judges didn't agree on everything. They disagreed once, in a way that reveals a real definitional gap between "factually true" and "grounded in this context." For a bounded scoring step like this, I'd reach for a typed-decision model over a second LLM call. I'd route anything under about 0.5 confidence to a human or a fallback LLM pass for the explanation. I just wouldn't have known any of that without running it myself.
