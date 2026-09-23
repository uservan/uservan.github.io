---
title: '[Research Preview] Where Does a Decision Model Break? A Needle-in-a-Haystack Test for Jev'
date: 2026-09-22
permalink: /posts/2026/09/jev-needle/
tags:
  - LLM
  - Jev
---
- authors: Wang Yang

> TL;DR
> - We built a controlled benchmark for Jev-class decision models (context + candidates → pick one) with three axes: how far the answer is, how many candidates there are, and how many history updates must be combined.
> - Jev 1.13.0 is perfect at retrieval and selection all the way to its 32k limit: 100% at 20k-token contexts, 64 candidates, any needle position.
> - It degrades only when it has to **integrate** several updates about the same entity into a current state: from 100% with 1 update to 53–63% with 16 updates. In 92% of its errors it picks a stale, earlier state.
> - Two follow-ups on that third case: give Jev a correct one-line state summary and it is back to 100%; give it a stale one and it follows the summary over the log 61–100% of the time. Asked whether a summary is correct, it catches a stale one only 62% of the time (chance 50%).
> - Practical takeaway for agent harnesses: compile the current state before calling Jev; do not hand it a raw history — and the correctness of that compiled state is the system's ceiling, because Jev neither repairs nor reliably checks it.

## Why

[Jev](https://docs.typesafe.ai/models) is a "System One" model: it reads a state and a bounded set of options and returns a choice with probabilities, in ~200 ms. In an agent loop it picks the next action every step. We wanted to know where its choices start to go wrong: when the context gets long, when there are many candidates, or when the relevant information is spread over several updates.

Existing benchmarks ([JevBench](https://github.com/fstandhartinger/jevbench)) use hand-written scenarios up to ~4k tokens, so the three factors cannot be separated. We generate the data programmatically and vary one factor at a time.

## Setup

Every item is the same shape: a context, a question, and 2–64 options; the model returns one option id. Answers are computed by a program, never by a model. Three heatmaps share the context-length axis (1k → 20k tokens, 28k for natural text; lengths measured with `o200k_base`, Jev's own count is ~1.45× for these records).

| Heatmap | Context filler | Needle | Other axis |
| --- | --- | --- | --- |
| **① Retrieval** | other users' records, or Paul Graham essays | one record `The access code of user_28195 is code_31340.` | needle position 10% / 50% / 90% |
| **② Selection** | other users' records | 64 records scattered at random (target + 63 candidate users) | number of options 2 → 64 |
| **③ State tracking** | other users' day-ordered project logs | the target user's *k* log lines: `Day 38: user_17824 joined project_H.` | updates *k* = 1 → 16 |

Heatmap ③ asks two questions on the same log: *event lookup* ("which project did user_17824 join on day 188?", one line to find) and *final state* ("after the last entry, which projects does user_17824 belong to?", *k* lines to find and merge). Wrong options for final state are the user's earlier states, states with one update skipped, or another user's state. Only legal operations are generated; every user starts with no projects.

Each cell has 10 independent scenarios × 3 option shuffles. The same scenario is reused across all cells with nested contexts (a shorter context is a prefix of a longer one), so cells differ only in the tested factor. Wrong options are always values that occur in the context; ids are random 5-digit numbers with no near-duplicates of the target. 7,650 requests including the two follow-ups, ~95M Jev input tokens, about $4.

Code and data: [github.com/uservan/jevneedlebench](https://github.com/uservan/jevneedlebench).

## Results

### ① Retrieval and ② selection: no degradation

All 75 cells are at 100% with the correct option at probability 1.0, including 20k-token contexts (≈30k Jev tokens, close to its limit), 64 options, needles in the middle, and both haystack types. Natural-text haystacks (essays) reach 28k tokens, also at 100%.

<img src="/images/post/jev_needle_h1_retrieval_kv.png" width="600">

<img src="/images/post/jev_needle_h2_selection.png" width="600">

### ③ State tracking: finding is fine, merging is not

Event lookup (one line) is 100% in every cell. Final state (merge *k* lines) falls with both axes:

| updates \ context | 1k | 2k | 4k | 8k | 16k | 20k |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |
| 2 | 1.00 | 1.00 | 0.93 | 0.80 | 0.60 | 0.93 |
| 4 | 1.00 | 1.00 | 1.00 | 0.90 | 0.80 | 0.73 |
| 8 | 1.00 | 0.73 | 0.77 | 0.73 | 0.77 | 0.70 |
| 16 | 0.80 | 0.63 | 0.60 | 0.53 | 0.63 | 0.57 |

<img src="/images/post/jev_needle_h3_log_event_lookup.png" width="600">

<img src="/images/post/jev_needle_h3_log_final_state.png" width="600">

The confidence drops with the accuracy: mean probability on the correct option goes from 1.00 (1 update) to about 0.5 (16 updates).

## Two follow-ups on the third case

Both reuse the 900 final-state items and append one line to each log, written by the generator (not by a model):
`Summary as of the last entry: user_17824 currently belongs to: project_A, project_E.` The *correct* version states the
true final set; the *stale* version states the set before the last update, the most common way a summarizer fails.

**Does Jev use a compiled state?** Same final-state question, summary line appended.

| summary | accuracy (all 30 cells) | note |
| --- | --- | --- |
| correct | **100%**, p ≈ 0.97 | every cell that was 53–80% without the line is back to 100% |
| stale | follows the stale summary 61–100% of the time when it is among the options | 68% at 1k tokens → 93% at 20k: the longer the log, the less it checks it |

**Can Jev check a summary?** Same log and line, but the question is *is that summary correct?* with two options.

| summary | judged correctly | note |
| --- | --- | --- |
| correct | 91% | drops to 60–80% at 8–16 updates, where it also cannot compute the state itself |
| stale | **62%** (chance 50%) | at 2 updates in 16k–20k logs: 0–7%; even with 1 update it misses about half |

So on the state-tracking side Jev is limited in all three roles: it does not merge history, it trusts a summary over the
raw log, and it cannot reliably verify that summary. It is a precise selector over a state someone else has compiled.

## Observations

1. **Retrieval is not the bottleneck.** Distance, position, candidate count and haystack type make no difference within Jev's context window. A single fact is found and selected every time.
2. **Integration is.** With one update the final state *is* the one line, and Jev is perfect. From two updates on it must combine lines, and accuracy drops with the number of updates faster than with context length.
3. **The errors are stale states.** Of 145 wrong final-state answers, 134 (92%) are an earlier state of the same user, and 137 are smaller than the correct set: Jev reads the first updates and misses later ones. It almost never confuses users (3 cases).
4. **This is a division-of-labor result, not a defect.** Jev is built for immediate judgment. The test says: whatever needs history to be merged must be merged *before* Jev sees it — and checked by something other than Jev.

## What this means for agent harnesses

In a loop like *propose → Jev picks → execute → summarize → memory*, the memory handed to Jev should be a **state snapshot** (current page, logged in, cart has 2 items, pending steps, recent failures), overwritten each step — not an appended list of what happened. A list of events about the same object is exactly the final-state task above, and it starts failing at two updates. A snapshot is the one-update case, which is at 100% up to the context limit.

## Limits

- 10 scenarios per cell; 95% intervals on the final-state cells are ±0.2, so trends are reliable and individual numbers are not.
- The summary lines are program-generated. How often a real summarizer LLM produces a stale state is a separate measurement, not done here.
- Only legal, non-conflicting operations; only set membership as the state; no similar-looking ids as distractors. These are the next things to add.
- Retrieval and selection hit the ceiling everywhere, so this data does not locate Jev's retrieval limit — only shows it is beyond 20k tokens and 64 options.
- This measures controlled retrieval and state tracking, not real multi-step execution.
