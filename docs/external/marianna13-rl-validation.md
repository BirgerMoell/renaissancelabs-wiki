# Validating RL environments (by Marianna13)

!!! note "External resource, not from Renaissance Labs"
    This is an external resource that Renaissance Labs considers relevant to its work. It was made by **Marianna13** and published under the LAION organisation on Hugging Face. Renaissance Labs did not make it, and the protocol and datasets are not ours. It is listed because it is relevant to those building RL environments. Questions and corrections go to the original author.

**Author:** Marianna13 (individual contributor)
**Protocol:** [Agentic RL dataset validation protocol](https://gist.github.com/marianna13/f9bc94d2ecb39c302c6c9b540ea6100a)
**Datasets:** [OT Validated RL datasets](https://huggingface.co/collections/laion/ot-validated-rl-datasets) (collection on Hugging Face, `laion`)

The idea is to check that tasks, environments and rewards work before any model is evaluated or trained on them, so that infrastructure faults are not mistaken for model failures. The collection describes its contents as RL datasets, mostly agentic, validated with oracle solutions.

## The validation protocol

The protocol has two parts: validating the dataset, then evaluating a model on it.

### Part 1: dataset validation

1. **Freeze the contract before running anything.** Record the dataset source and immutable revision, the task-ID manifest and how tasks were selected, the expected oracle and negative-control rewards, the execution profile (backend, architecture, network policy, privilege level, mounts, dependency policy) and the minimum acceptable task count. Validation on one profile is not proof for another: success under Docker does not show that a task works offline under rootless Apptainer.
2. **Run a representative pilot.** Pick tasks across repositories, image types, dependency stacks and verifier types, and run two agents on each:
    - an **oracle** (reference) agent, which must get the expected passing reward;
    - a **negative control** (an empty or known-wrong agent), which must not pass. Partial-credit behaviour is checked where the dataset defines it.

    Fixes for infrastructure incompatibilities must not weaken tests, change task meaning or leak solution information, and every patch is versioned.
3. **Validate the full candidate pool.** Run one oracle attempt on every candidate task with the production configuration. Each non-pass is classified as infrastructure failure, invalid task or verifier, unsupported execution contract, or a valid task whose oracle fails. The exception-free passes are frozen if they meet the predeclared minimum, and the negative control is then run on exactly that frozen set. A task is compatible only if the oracle passes and the control is rejected, with no infrastructure exceptions.
4. **Preserve evidence.** Keep source revision, task IDs and selection seed, manifest hashes, per-task oracle and control rewards, runtimes and checksums, image hashes and task-to-image routing, patch provenance, logs, and an exclusion ledger with one reason per excluded task. Failed and superseded runs are kept, and counts are reported with explicit denominators.
5. **Package and re-test.** Materialise the released package in a fresh location and repeat a small oracle/control smoke test through the documented execution profile.

A dataset, or a declared compatible subset, passes when every retained task has an exception-free oracle pass and an exception-free negative-control rejection, the count meets the minimum, all exclusions and patches are documented, and the fresh-materialisation smoke test passes. Results are reported as scoped evidence, for example "N tasks passed on a stated machine, architecture and runtime", not as a general claim.

### Part 2: model evaluation after infrastructure validation

Once the dataset passes on the intended profile, the exact model checkpoint is evaluated before any RL update, with model revision, agent implementation, prompts, tool schema, sampling settings, limits, timeouts and reward code all frozen to match the planned RL setup.

- **Deterministic pass@1** over the full frozen set gives the baseline and exposes systematic model or harness failures.
- **Sampled pass@k** (commonly eight samples per task) gives mean sample accuracy and task-level pass@k. The gap between them measures exploration headroom. Groups whose rewards have zero variance are also reported, since they give no useful within-group RL signal.
- Per-task and per-family rewards, turns and token use, latency, tool-call failure rates, limit and timeout terminations, malformed tool calls and environment or verifier errors are recorded.
- Representative successful and failed trajectories are read, and failures are classified as infrastructure, invalid task or verifier, harness, genuine capability, or uncertain. A defect found this way is fixed, versioned and revalidated, and the affected evaluation is rerun. Tasks are never removed just because the model fails them.

The baseline and per-task outputs are kept so later checkpoints can be compared under exactly the same contract.

## Validated datasets

All datasets below are in the [collection](https://huggingface.co/collections/laion/ot-validated-rl-datasets). Counts and licences are as stated on each dataset card when we checked. Validation scope differs between datasets, so read the "Validation" column before using one. Most are validated for **offline, rootless Apptainer on aarch64**, and the cards say that other backends need their own checks.

| Dataset | Source | Tasks | Validation | Licence |
|---|---|---|---|---|
| [swegym-1k-rootless-apptainer](https://huggingface.co/datasets/laion/swegym-1k-rootless-apptainer) | SWE-Gym | 1,029 | Oracle passed and no-op rejected on all tasks, no exceptions | Apache-2.0 |
| [r2egym-1k-rootless-apptainer](https://huggingface.co/datasets/laion/r2egym-1k-rootless-apptainer) | R2EGym | 1,145 | Oracle and no-op, all tasks, no exceptions; fresh-materialisation smoke test passed | Apache-2.0 |
| [endless-terminals-rootless-apptainer](https://huggingface.co/datasets/laion/endless-terminals-rootless-apptainer) | Endless Terminals | 1,000 | Oracle and no-op, three attempts each; 25 tasks use host network or a proxy and are not strictly offline | MIT |
| [rts-1k-rootless-apptainer](https://huggingface.co/datasets/laion/rts-1k-rootless-apptainer) | Recursive Task Synthesis | 984 | Strict passing subset of a 1,000-task cohort; 16 failures kept in an exclusion ledger | CC BY 4.0 |
| [nemotron-terminal-easy-clean1k-rootless-apptainer](https://huggingface.co/datasets/laion/nemotron-terminal-easy-clean1k-rootless-apptainer) | Nemotron Terminal synthetic tasks | 1,000 | Oracle (replayed from completed traces) and no-op, three attempts each | CC BY 4.0 |
| [tmax-legacy-1k-rootless-apptainer](https://huggingface.co/datasets/laion/tmax-legacy-1k-rootless-apptainer) | TMax-15K (legacy subsets) | 1,000 | Oracle 3000/3000 and no-op 3000/3000; gold solutions hash-pinned but not agent-visible | Upstream ODC-By; none set on the card |
| [tasktrove-bugsinpy-v4-oracle17-apptainer-v1](https://huggingface.co/datasets/laion/tasktrove-bugsinpy-v4-oracle17-apptainer-v1) | TaskTrove BugsInPy v4 | 17 of 479 | Oracle-verified subset only; the full source is not claimed as verified | Mixed ("other") |
| [openthoughts-tblite-dev-apptainer-v1](https://huggingface.co/datasets/laion/openthoughts-tblite-dev-apptainer-v1) | OpenThoughts-TBLite | 53 validated of 100 | 27 have no gold solution (grader rejects empty attempt only); 20 cannot run rootless | Apache-2.0 |
| [terminal-bench-2-1-apptainer-v1](https://huggingface.co/datasets/laion/terminal-bench-2-1-apptainer-v1) | Terminal-Bench 2.1 | 74 validated of 89 | 15 excluded; scores are not comparable to the full leaderboard | Apache-2.0 |
| [tasktrove-curriculum-easy-rootless-apptainer](https://huggingface.co/datasets/laion/tasktrove-curriculum-easy-rootless-apptainer) | TaskTrove curriculum (easy) | 505 | Runtime-tested only; source has no reference solutions, so not oracle-certified | Apache-2.0 |
| [tasktrove-curriculum-medium-v2-rootless-apptainer](https://huggingface.co/datasets/laion/tasktrove-curriculum-medium-v2-rootless-apptainer) | TaskTrove curriculum (medium v2) | 489 | Runtime-tested only; not oracle-certified | Apache-2.0 |
| [inferredbugs-rootless-apptainer](https://huggingface.co/datasets/laion/inferredbugs-rootless-apptainer) | InferredBugs | 9,991 | Runtime-tested only; 7,706 offline-only verifier patches (test logic unchanged) | Other |

The first five and the TMax set are the strongest claims: every task has both an oracle pass and a negative-control rejection. The last three rows are weaker and say so on their own cards.

## Caveats

- The task counts in the TBLite row (100 total) are our arithmetic from the card's 53 + 27 + 20; check the card for the current figures.
- Validation is scoped to the stated execution profile, mostly a particular supercomputer with rootless Apptainer. It says nothing about Docker or other cluster setups.
- Licences differ per dataset and some embed third-party code. Check each card before redistributing.
- We read the dataset cards and the protocol; we did not re-run any validation.

## Related

- [RL rollouts with internet access](../projects/rl-rollouts-with-internet.md): our own notes on running rollouts safely on a cluster.
