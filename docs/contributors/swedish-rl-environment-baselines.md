# Verifiable Swedish RL environments and baselines (Birger Moëll)

!!! note "Contributor write-up"
    Author: **Birger Moëll**. The measurements ran on a shared GPU cluster. This page was drafted with AI assistance (Claude Code) from the project notes and experiment logs, and we have not re-run them.

**Status:** environments built and baselined. RL training on top of them is in progress and not reported here.

Reinforcement learning pays off where an answer can be checked by code. This project builds Swedish environments of that kind, where each task comes with a generator and a verifier, and measures how current open models do on them before any training.

## The environments

Three groups are covered here, built from real Swedish source text.

- **Law (`juridik`).** Deadlines under the administrative procedure act, citing the correct provision, and choosing the right decision type. The text comes from parsed statutes (17 laws).
- **Finance (`finans`).** Late-payment interest, VAT, and credit calculations.
- **Medicine (`medicin`).** Patient data law questions, medical certificates, and plain-language rewriting of clinical text.

Every task is scored by a verifier that parses the answer and checks it against the source. All environments are decision support for a human, and the training data is synthetic.

## Baseline across open models

The same 1,080 tasks (27 environment and difficulty cells), scored as mean reward from 0 to 1 with reasoning off and on.

| Model | Reasoning off | Reasoning on |
|---|---:|---:|
| MiMo-V2.6-Flash-RL | 0.67 | 0.72 |
| GLM-5.3-Flash | 0.55 | 0.70 |
| Nemotron 3.5 Lightning | 0.50 | 0.63 |
| Qwen3.6-27B, Gemma-4-31B | about 0.50 | runs crashed |

GLM ignored the reasoning-off setting and truncated 40% of its answers. For the Qwen3.6 and Gemma runs the server crashed with reasoning on, and the cause was not investigated.

## Where the headroom is

A first baseline on a separate Qwen model showed which environments are saturated and which are not.

- **Saturated (about 0.9 to 1.0):** the medical certificate and patient-data tasks, VAT, and the easiest citation tasks. These are not worth training on until they are made harder.
- **Real headroom:** deadline computation (scores 0.07 to 0.37, with the model mis-adding dates and inventing paragraph numbers), choosing a decision type (the model over-chooses "communicate" and invents chapters), and credit calculations that need reasoning.
- **Truncation:** plain-language rewriting with reasoning on was cut off 61 to 74% of the time.

## Lessons for building verifiers

These cost time and are the most reusable part of the work.

- Parse the answer only from the text after the reasoning block. The first version picked up draft JSON from inside the reasoning.
- Score truncated answers as zero.
- Accept legitimately extra citations, such as cross-references inside quoted statute text.
- Test shortcut baselines. An all-yes and an all-no answer scored 0.55 before the task was rebalanced.
- Re-score stored samples when a verifier changes, instead of re-running the GPUs.

## Other environments

The project also has multi-turn environments: a search task over a fictional offline Swedish web with injected distractions, and code-writing tasks with hidden tests. Running model-written code is done only inside a restricted sandbox, and only with explicit approval. A separate set of text-to-SQL environments (single-shot and multi-step agents) was baselined too: a raw model scores 0.67 to 0.84 on query shapes it has seen and 0.15 to 0.24 on new ones.

## Related

- Public environments: [Svenska RL](https://github.com/BirgerMoell/svenska-rl).
- [Small-model RL experiments on a laptop](small-model-rl-experiments.md), the earlier series.
