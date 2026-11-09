# Small-model RL experiments on a laptop (Birger Moëll)

!!! note "Contributor write-up"
    Author: **Birger Moëll**. A series of small experiments run on a laptop. This page was drafted with AI assistance (Claude Code) from the experiment reports kept with the code, and we have not re-run them.

**Status:** complete as a series. Mostly negative results, which is the point of writing them down.

The question behind these runs was whether reinforcement learning with a verifier (RLVR) can improve a small model on Swedish tasks whose answers can be checked by code. The runs used 4-bit quantised Nemotron 3 Nano models (30B-A3B and 4B) and an LFM2.5 model on a 32 GB Apple Silicon machine, with LoRA adapters and the MLX framework. Every task has an exact or rule-based verifier, and the training sets are synthetic.

## Results

| Experiment | Task | Result |
|---|---|---|
| Nemotron Nano 30B, math | Swedish slice of an outcome-verified math set | 48 and 240 steps: no held-out gain (6/24 before, 5-6/24 after) |
| LFM2.5-8B-A1B | Small hand-written Swedish questions | 24 steps fell from 1/8 to 0/8 exact answers |
| Casework, Nemotron 30B | Synthetic intake and grant recommendations, 33 held-out cases | SFT raised mean reward 0 to 0.162; RL raised it to 0.206 but grant decision accuracy fell 9/21 to 7/21 |
| Casework, Nemotron 4B | Same cases | SFT plus 48-step RL: mean reward 0.203 to 0.292 to 0.415; 8/21 grant decisions correct |
| Curriculum, 4B | Short Swedish decision lessons | Easy-decision accuracy 0/40 to 38/40; original-case reward 0.415 to 0.600; eligible grants still all denied |
| Counterfactual grants, 4B | Paired facts and approval limits, 65 challenge cases | Decision accuracy 20/65 to 25/65, but 0/35 eligible cases approved; original-case accuracy regressed 16/21 to 12/21 |
| Exact-reward ladder | Fact reading, missing documents, ledger arithmetic, decision priority | SFT and RL both 0.267, with 0/20 full solutions |
| Ledger components | Four decomposed ledger tasks, 170 held-out cases | Sum accuracy 28/80 base, 31/80 SFT, 29/80 after RL |
| Rule priority | Fictional ordered rules, one-word answers, 120 held-out cases | Base 54/120; 96-step RL 55/120; 240-step low-rate RL 44/120; high-rate run collapsed to a single answer (33/120) |

## What we learned

- **SFT helps; RL on top did not reliably help.** Supervised fine-tuning on correct examples produced the large gains (for example 0/40 to 38/40 on easy decisions). Small RL runs added little or nothing on held-out cases.
- **Partial reward hides the real result.** In the casework runs the mean verifier reward rose while the accuracy of the final decision stayed flat or fell. Report them separately.
- **Many steps carry no signal.** In several runs more than half of the RL steps were skipped because every sample in a group got the same reward, so there was no advantage to learn from.
- **A high learning rate collapses the policy.** One run ended answering the same word for all 120 cases.
- **Counterfactual data can trade one skill for another.** Further SFT on paired examples improved the new challenge set and damaged the original grants.
- **The setup was limited.** The local trainer had no active KL penalty or clipped policy ratio, the models were 4-bit, and the test sets were small. A better-specified trainer and larger held-out sets are the next step, not a conclusion that RL cannot work here.

## Related

- [Verifiable Swedish RL environments](swedish-rl-environment-baselines.md), the larger-scale follow-up on a cluster.
