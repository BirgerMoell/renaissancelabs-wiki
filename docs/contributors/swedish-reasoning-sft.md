# Swedish reasoning for Nemotron 3.5 Lightning: an SFT experiment (Birger Moëll)

!!! note "Contributor write-up"
    Author: **Birger Moëll**. The experiment ran on a shared GPU cluster. This page was drafted with AI assistance (Claude Code) from the saved experiment outputs, and the evaluation figures below were computed from those saved outputs for this page. No human has re-run them.

**Status:** one LoRA run per variant, evaluated once. Treat the differences as indicative.

A hybrid reasoning model such as Nemotron 3.5 Lightning "thinks" before it answers, and for a Swedish question it thinks in English. This experiment asks whether the reasoning can be moved into Swedish with a small supervised fine-tune, and what that costs in accuracy.

## Method

1. **Prompt pool.** Swedish prompts in three groups: math, general questions, and medical multiple choice (MedQA translated to Swedish).
2. **Traces.** A teacher model, run with Swedish reasoning forced, wrote a reasoning trace and an answer for each prompt. Two teachers were tried: Nemotron itself and Gemma.
3. **Filtering.** A trace was dropped if its final answer was wrong, if the reasoning or answer was not in Swedish (checked with a word-list language test), if it merely copied the prompt, or if it was unclosed or degenerate.
4. **Data.** 4,923 training examples from the Nemotron teacher (math 1,158, general 2,135, medical 1,150, plus 480 English examples as replay to protect English behaviour). The Gemma-teacher set has 4,194.
5. **Training.** LoRA (rank 128) on the Nemotron 3.5 Lightning BF16 model, 614 steps over two epochs, about 3 GPU-hours on four H200 GPUs.

The filters mattered. For the Nemotron teacher, 2,013 of the traces were unclosed or degenerate, 324 had reasoning that was not Swedish, and 175 math and 606 medical traces had wrong answers.

## Evaluation

Held-out prompts: 250 Swedish math problems, 150 Swedish medical questions with gold answers, 40 English prompts, and a few Swedish civic questions. The model gets no instruction about language, thinking is on, temperature 0.6, and replies are capped at 6,000 tokens. "Swedish reasoning" means at least 80% of the reasoning words are Swedish by the word-list test.

| Model | Swedish reasoning | Math correct | Medical correct | Truncated (medical) |
|---|---:|---:|---:|---:|
| Base (Lightning) | 0% | 246/250 | 111/150 | 15/150 |
| SFT, step 300 | 100% (math), 96% (medical) | 239/250 | 112/150 | 22/150 |
| SFT, step 614 | 100% (math), 97% (medical) | 239/250 | 116/150 | 17/150 |
| SFT, Gemma-teacher data | 100% | 241/250 | 111/150 | 10/150 |

For reference, two other open models with reasoning on, using the same prompts: GLM-5.3-Flash 222/250 math and 133/150 medical, MiMo-V2.6-Flash 188/250 and 126/150. Neither reasons in Swedish.

## What the results show

- **The language switch works.** After training, the reasoning is Swedish for essentially every Swedish prompt, where the base model was at 0%. English prompts still get English reasoning, so the switch follows the language of the prompt.
- **Math cost a few points.** Math accuracy moved from 98.4% to 95.6% (246 to 239 of 250). The sample is small and the run is single, so this is a small, plausible cost, not a measured one.
- **Medical accuracy did not drop** (111 to 116 of 150), though the translated questions are themselves noisy.
- **Truncation is a real constraint.** Between 10 and 22 of the 150 medical replies hit the token cap, before and after training. Comparisons across checkpoints should be read with that in mind.
- **Reasoning length.** Swedish reasoning is somewhat longer on math (a mean of about 250 words against 180 for the base), as Swedish needs more tokens for the same content.
- **An early checkpoint was bad.** The step-100 checkpoint dropped to 45/150 on the medical set while its reasoning was already Swedish, so the language switch arrives well before accuracy recovers.

## Caveats

- One run per condition, one seed, small test sets, no confidence intervals.
- Language is identified with a word-list heuristic, not a trained classifier, and answers are scored by extracting a boxed answer or a letter.
- The medical questions are machine-translated, and the translation was not reviewed by a clinician.
- The same SFT recipe was also tried on an Apertus model. It reasoned in Swedish but was much less accurate and truncated far more often, so the recipe does not transfer without change.
- We have not measured whether Swedish reasoning improves anything downstream, such as answer quality for Swedish readers. That is the open question this experiment leaves.

## Related

- [Training Nemotron 3.5 Lightning for Swedish](../ideas/swedish-nemotron-first-model.md), which plans a controlled version of this comparison.
