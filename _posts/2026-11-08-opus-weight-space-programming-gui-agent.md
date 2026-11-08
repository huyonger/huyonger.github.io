---
title: "Claude Opus 5.5 Does Weight-Space Programming on a GUI Agent"
date: 2026-11-08
tags:
  - weight editing
  - interpretability
  - GUI agents
  - AndroidWorld
---

Can an LLM make a GUI agent better by editing the agent's weights directly, with no training at all? We had Claude Opus 5.5 analyze the failures of Qwen3.5-0.8B in AndroidWorld and fix them in weight space. Changing 4,021 numbers in the output layer took fully formatted steps from 30% to 99% and nudged task success from 13.5% to 17.5%. The bigger lesson was about what such edits can and cannot do: they fix bugs, not missing skills.

{% include toc icon="list" title="Contents" %}

## TL;DR

- Inspired by *Sparks of Weight-Space Programming* (MIT CSAIL), we used Claude Opus 5.5 to improve a small GUI agent, Qwen3.5-0.8B on AndroidWorld, by editing its weights directly: no gradients, no fine-tuning.
- In 348 episodes of the agent's own trajectories, 70% of steps broke the required output format. Most often the model skipped the `<summary>` line that the agent's memory is built from.
- Each format error traced back to a single token choice, often a near tie between the right token and a wrong one. We fixed four such choices by editing four rows of the output layer: 4,021 numbers, 0.00046% of the model.
- Over three AndroidWorld runs, fully formatted steps rose from 30.3% to 99.1% and success from 13.5% to 17.5% on our harness metric. The gain on the official verifier (16.7% to 19.3%) is not yet statistically significant, and episodes killed by thinking loops went up, from 25 to 43.
- The edits fixed what the model already knew how to do. They could not teach it where to tap.

## Background

In early November, Rikiya Takehi, Jayden Teoh, Alex Zhang and Omar Khattab (MIT CSAIL) published [Sparks of Weight-Space Programming](https://rikiyat.github.io/blog/weight-space-programming/). They gave a frontier LLM, GPT-6 Astra, tools to read and write another network's weights and asked it to change that network's behavior directly in weight space. One experiment is especially relevant here. Qwen3.5-0.8B often gets stuck in reasoning loops on math problems. Astra found a "loop direction" in the model's activations, traced it to two MLP layers, and subtracted it from 131,072 weights (0.016% of the model). MATH-500 went from 7.9% to 48.1%, with no training.

I work on GUI agents, so the natural question was whether the idea transfers: can we make a small phone-operating agent better by editing its weights? I ran the project over two days with Claude Opus 5.5 (in Claude Code) as a hands-on research agent. It read the trajectories, wrote the analysis and editing code, ran experiments on our GPUs and iterated on the edits. I set the goals, made the calls and ran the AndroidWorld evaluations.

## Setup

- **Model:** Qwen3.5-0.8B, thinking on.
- **Benchmark:** AndroidWorld, 116 tasks, up to 50 steps per episode.
- **Agent:** single-turn. At each step the model sees the system prompt, the task, a history and the current screenshot, and must reply in exactly this format:

```text
<think> … </think>
<summary>I need to open Settings to access the brightness slider.</summary>
<tool_call>{"action_type":"click","coordinate":[870,829]}</tool_call>
```

Coordinates are on a 0–1000 grid. The history is simply the list of the model's own past `<summary>` lines, so a step without a summary leaves a blank line: the agent forgets that step. A reply without a usable action ends the episode as an error.

We report two success measures. The **verifier** is AndroidWorld's official check of the phone's final state. The **harness** metric also requires the episode to end normally rather than be aborted for invalid output, so it is never higher than the verifier.

The baseline is three clean runs of the unedited model, 348 episodes and about 12,400 steps:

| Baseline (3 × 116 episodes) | |
|---|---|
| Verifier success | 16.7% (58/348) |
| Harness success | 13.5% (47/348) |
| Fully formatted steps | 30.3% |

## Step 1: Read the trajectories

Before touching any weights, we needed to know what goes wrong. The main causes among the 290 failed episodes:

| Failure | Episodes |
|---|---|
| Used up all 50 steps, repeating the same actions | 144 |
| Ended by giving a wrong answer | 56 |
| Aborted: the reply had no valid action (format error) | 33 |
| Aborted: thinking looped until the 16,384-token limit | 25 |

Three patterns stood out.

1. **It can't reach the right app.** By a text heuristic over the model's own reasoning, failed episodes spend half their steps still trying to open the target app, against 24% in passed episodes. The model rarely opens the app drawer and sometimes taps the wrong icon.
2. **It breaks the output format.** Only 30% of steps follow the required format. 67% have no `<summary>`, which wipes the agent's memory of those steps. Others leave the tool call unclosed or add stray characters after the JSON.
3. **It taps a placeholder.** 9.3% of all taps land on exactly `[500,500]`, the example coordinate in the prompt's action table.

## Step 2: Bug or missing skill?

These failures are not the same kind of problem. Some are **bugs**: the model can do the right thing, and often does, but slips. Others are **missing skills**: the model doesn't know how. The MIT authors make the same point about their own result. Fixing loops, they write, "lets the model finish answers it could already work out", and on a model that rarely loops (Gemma 3 270M) their edits did not help.

The format errors looked like bugs: the same model writes a perfect summary at one step and skips it at the next. Getting lost and tapping placeholders looked like missing skills. So the format became the main target.

## Step 3: Fix the format

### First attempts: edit the middle of the network

We started the way most activation-editing work does. Replay recorded steps, collect hidden states where the model writes a summary and where it skips one, take the difference of the means as a "skip direction", and remove that direction from the matrices that write into the residual stream (directional ablation).

At layer 16 this worked: the model wrote a summary on 100% of held-out steps. But there were side effects. Usable actions fell from 98% to 88%, and more replies ran past the token cap of our test (9 of 120, against 1 of 120). Ablations at other layers broke the action JSON.

We also tried the MIT team's released loop-fix model on our agent. It did fix thinking loops on our hardest steps (looping replies fell from 18 of 87 to 2 of 87), but it wrecked the agent's behavior: summaries dropped from 46% to 4%, usable actions from 98% to 87%, and strict format compliance to 0%. A direction that is clean on math problems turned out to be entangled with writing summaries.

### Find the fork

The breakthrough was to stop asking *where in the network* the error lives and ask *where in the reply* it happens. A language model writes one token at a time, so every format error has a fork: a position where the right token and a wrong one compete. Replaying replies position by position showed four forks:

| Position | Right token vs. wrong token | Unedited model |
|---|---|---|
| Right after `</think>` | `<` (starts `<summary>`) vs. `<tool_call>` | A near tie, 0–2 logits apart |
| Right after `</summary>` | Newline vs. end of turn | End of turn sometimes wins |
| After the last coordinate digit | `]}` vs. a lone `]` | A lone `]` starts 60% of broken endings |
| Right after the action JSON | `</tool_call>` vs. stray characters | Stray characters occasionally win |

The third fork was a surprise. We first assumed the junk started after the closing `}`, and edits there did nothing. Only token-by-token replay showed that the real fork comes one step earlier: once the model picks a lone `]` instead of `]}`, a run of stray braces and backticks almost always follows.

### Edit only the rows that decide

At every position, the model scores each token `t` as the dot product of its final hidden state `h` with that token's row `e_t` in the output layer. So "summary or no summary" is literally a comparison of two rows against `h`, and the weights to change are just those rows. To make an edit act only at its fork, we add a direction to the row instead of shifting the whole row:

```text
score(t) = h · e_t

e_t  ←  e_t + c·u        new score = h · e_t + c·(h · u)
```

Here `u` is the normalized difference between the mean hidden state at the fork and the mean at nearby positions that must not change. For the summary fork, `h · u` averages +42 right after `</think>` and −40 right after `</summary>`, with no overlap. So the edit is strong where it should act and silent elsewhere.

Because the score is linear in the row, the effect of any strength `c` can be computed exactly from stored hidden states, without rerunning the model. We picked the smallest `c` that fixed at least 99% of target positions with a margin of 1–3 logits, while flipping none of the clear decisions at the other forks or at about 50,000 ordinary positions.

One more detail mattered. Qwen3.5-0.8B ties its output layer to its input embeddings, so editing the `<tool_call>` row also changed how the model *reads* `<tool_call>` in the system prompt. With tied weights, the model started ending its turn right after the summary (8.5% of steps became 30%). So we first untied the output layer, an exact copy whose outputs are bit-identical to the original, and edited only the copy.

The final model changes four rows:

| Token row | Edit | Fixes |
|---|---|---|
| `<tool_call>` | Lower right after `</think>` | Skipped summary |
| End of turn | Lower right after `</summary>` | Stopping before the action |
| Lone `]` | Lower after the last coordinate digit | Stray characters after the JSON |
| `</tool_call>` | Higher right after the JSON | Unclosed tool call |

(In Qwen's vocabulary the end-of-turn token is `<|im_end|>`.) That is 4,021 changed numbers, 0.00046% of the model's 873 million parameters. (Four rows hold 4 × 1,024 values; 75 of the edits were small enough to round back to the original bf16 value.) All 488 original weight tensors are bit-identical. The price of untying is a separate 254-million-parameter output layer, which grows the checkpoint from 1.75 GB to 2.26 GB. The inference code doesn't change: vLLM loads the edited model as is.

## Results

### Offline checks

Before the live runs, we checked the edited model on 180 held-out steps from tasks that were not used to build the edits:

| | Unedited | Edited |
|---|---|---|
| Strict format, greedy decoding | 37% | 97% |
| Strict format, sampling (T = 0.6) | 40% | 95% |
| Usable actions | 98% | 97–98% |
| Same greedy action as the unedited model | — | 75% |

The mix of action types stayed close to the unedited model's. Thinking loops on the hardest steps stayed at the unedited level: not fixed, not worse.

### AndroidWorld

Three full runs, with the same harness, prompt and step budget as the baseline:

| | Unedited | Edited |
|---|---|---|
| Fully formatted steps | 30.3% | **99.1%** |
| Harness success | 13.5% | **17.5%** |
| Verifier success | 16.7% | 19.3% |
| Episodes aborted for format errors | 33 | 18 |
| Episodes aborted by thinking loops | 25 | **43** |

Per run, harness successes went from 18, 12 and 17 tasks to 20, 19 and 22: the worst edited run beat the best unedited one. Paired by task, the harness gain is +4.0 points (95% CI 0.0 to +8.0) and the verifier gain is +2.6 points (95% CI −1.4 to +6.6). So the format is fixed and success went up a little, but more runs are needed before calling the gain real on the official metric.

A modest gain is what we expected. Format was never the main bottleneck; getting lost was. The gain most likely comes from two side effects of a fixed format: fewer episodes aborted for invalid output, and a working memory, since every step now adds its summary to the history.

The regression is real, though: episodes aborted by thinking loops rose from 25 to 43. Our current hypothesis is that a full history can hurt. 38% of consecutive steps now repeat the previous summary word for word, so when the agent is stuck, its history fills up with identical lines, which may push the model further into repetition. The offline checks could not catch this, because they replayed the unedited model's mostly blank histories.

## What didn't work

- **Removing the placeholder tap.** A direction found at layer 4 cut `[500,500]` taps on held-out steps from 96% to 9%, but it also moved 24% of normal taps, and the replacement taps were just other defaults: an icon in the dock, or the other example coordinate in the prompt. Most of the time the target wasn't even on screen. The placeholder is a symptom of not knowing where to tap, not a bug hiding a skill.
- **Fixing thinking loops with an output-row edit.** At the last layer, looping and normal states overlap, and inside a loop `</think>` trails the top token by about 24 logits. Any edit strong enough to end the loops also changed thousands of normal positions.

## How this compares with the MIT experiments

| | Sparks of Weight-Space Programming (math experiment) | This project |
|---|---|---|
| Who edits | GPT-6 Astra, autonomously, in a fixed loop with a budget of forward passes | Claude Opus 5.5 and me, interactively |
| Task | Math, single turn | AndroidWorld: multimodal, multi-step |
| Bug | Reasoning loops | Broken output format |
| Where | Two middle MLP layers (15 and 19), 64 columns each | Four rows of the output layer |
| Weights edited | 131,072 (0.016%) | 4,096, of which 4,021 changed (0.00046%) |
| Rules | Edits to the output layer, embeddings and final norm not allowed | Output layer edited, after untying |

Two caveats. First, ours was not an autonomous run. The method and the stopping criteria came out of a back-and-forth, so this project does not test the post's central claim that an LLM can do this on its own. Second, according to the released logs, the math experiment did not allow edits to the output layer, embeddings or final norm, most likely to block shortcuts that move the outputs without understanding the network. Our edits live exactly there, so the edit sizes (131,072 weights against 4,096) are not a like-for-like comparison. Each of our edits acts only at its own fork rather than biasing a token everywhere, but they are still output-side fixes.

## Takeaways

1. **Weight editing is debugging.** It fixes behaviors the model already has but executes unreliably. It does not add skills. On a small GUI agent, the bugs (format) were fixable with a tiny edit; the missing skills (finding apps, grounding taps) were not.
2. **A quick test for "bug or missing skill".** If the model sometimes gets it right, or the right token is a close second, an edit can probably fix it. If it never gets it right, you need training or a bigger model.
3. **Edit where the error is decided.** A wrong token choice is decided at the output layer, and fixing it there leaves the model's thinking untouched. Errors that live in the model's internal state, like thinking loops, need the middle-layer approach of the MIT work.
4. **Test in the loop.** Offline checks passed, yet the live agent looped more, because the fix changed the agent's own inputs: its history. Agents feed their outputs back to themselves, so evaluate edits on-policy.

## Looking ahead: debugging as a path to self-improvement

Weight-space programming is a genuinely new way to change a model, but in this project it behaved like debugging, not teaching. That may be worth more than it sounds. What a deployed model achieves is roughly what it is capable of, minus the bugs that keep it from using that capability, and for small agents the gap can be large. The MIT team's loop fix took MATH-500 from 7.9% to 48.1%; our format fix gave the agent a working memory on nearly every step instead of about a third of them. Fixing bugs is one way of making a model better.

It also suggests a different route to recursive self-improvement (RSI). Everything in this post is a loop that a model could run: analyze trajectories, find recurring failure patterns, sort them into bugs and missing skills, localize each bug, write a small edit, verify it in the environment, and keep or roll back the edit. Bugs would be patched in weight space, quickly and cheaply. Missing skills would go to a slower loop of data collection and training, fed by trajectories from a model that no longer wastes episodes on broken output.

For this loop to compound rather than stall, three things have to hold:

- **Verification must be on-policy and statistically sound.** Our offline checks passed, yet the live agent looped more. With run-to-run noise of several points, a loop that accepts edits on thin evidence will drift.
- **Every patch needs regression tests.** Fixing one bug changed the agent's inputs and surfaced another. Each new edit should be checked together with all earlier ones, like a test suite in software.
- **Edits must stay small and readable.** An edit of 4,021 numbers can be explained and reviewed one row at a time. That is what makes it reasonable to let a model modify itself, or a sibling model, under human oversight.

There are two honest limits. In this project a stronger model edited a much weaker one; a model debugging itself is a harder problem that is still open. A practical middle ground is a strong model acting as the debugger for a fleet of smaller deployed models. And bug fixing alone saturates once the bugs are gone: the loop keeps improving a model only if it is paired with training.

Our next steps follow from this: go after the thinking loops with middle-layer localization, using the 51 real loop cases the edited model produced; test the repeated-history hypothesis; and let an LLM run the whole loop autonomously, as in the original work, under the verification rules above.

*Thanks to the authors of Sparks of Weight-Space Programming for releasing their logs and edited model, which made a close comparison possible.*
