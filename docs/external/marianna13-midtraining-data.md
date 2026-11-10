# Mid-training and SFT data for a 9B model (by Marianna13)

!!! note "External resource, not from Renaissance Labs"
    This is an external resource that Renaissance Labs considers relevant to its work. It summarises a working document by **Marianna13** from her [prelude-posttrain](https://github.com/marianna13/prelude-posttrain) repository. Renaissance Labs did not make it. The figures are hers, taken from the document as it stood at its last update (23 October 2026), and we have not reproduced them. Corrections go to the original author.

**Author:** Marianna13 (individual contributor)
**Source:** [issue-0054-midtraining-data.md](https://github.com/marianna13/prelude-posttrain/blob/main/docs/issue-0054-midtraining-data.md)
**Status in the source:** the mid-training document was in progress when last updated (23 October 2026). A later document records SFT v2 as trained.

The goal is a reproducible, decontaminated mid-training mixture for the OpenEuroLLM Prelude 9B base model. It is used to extend the context window from 4K through 16K, 32K and 64K (and then 128K with YaRN) while keeping the base model's math and code ability.

## Data sources

Five core collections on Hugging Face, each pinned to an exact revision in the source, make up about 1 TiB of data and 825 billion tokens before filtering.

| Collection | Contents | Tokens (seen) |
|---|---|---:|
| [Nemotron-Cascade-2-SFT-Data](https://huggingface.co/datasets/nvidia/Nemotron-Cascade-2-SFT-Data) | Math, science, software-engineering agent, conversational agent, chat, instruction following | 126B |
| [Nemotron-Pretraining-Specialized-v1](https://huggingface.co/datasets/nvidia/Nemotron-Pretraining-Specialized-v1) | Reasoning, question answering, STEM, textbooks, scientific coding | 297B |
| [UltraData-Stack-v3-L2](https://huggingface.co/datasets/openbmb/UltraData-Stack-v3-L2) | Code files in 11 languages with quality scores | 230B |
| [stack-v3-train](https://huggingface.co/datasets/HuggingFaceCode/stack-v3-train) | Repository-level code | 165B |
| [Nemotron-Terminal-Corpus](https://huggingface.co/datasets/nvidia/Nemotron-Terminal-Corpus) | Terminal-agent conversations | 7B |

A bounded proxy for general-text replay ([DCLM](https://huggingface.co/datasets/mlfoundations/dclm-baseline-1.0), Nemotron-CC, and [FinePDFs](https://huggingface.co/datasets/HuggingFaceFW/finepdfs) in English and 23 other EU languages) is added to limit forgetting, plus a separate set of very long PDF documents (at least 32K tokens) for the long-context stages. Some components were excluded on purpose: the Wiki Rewrite subset, the agent subset that duplicates the terminal corpus, and a local code aggregate with unclear file-level provenance.

**Code selection.** Repositories need at least 8,192 tokens, a match in the quality-scored set, and at least 20% scored files, and are ranked by density and quality. The author chose to include both permissively licensed and unlicensed ("no_license") code, which gives 128,122 repositories and 5.9B tokens, against 3,439 repositories and 0.26B tokens for permissive-only. This is a licensing decision that others may want to make differently.

## How the data is prepared

1. **Freeze and census.** Every file is checksum-verified, and token counts use the frozen tokenizer and chat template. Records with invalid schemas are rejected and logged (46 in total).
2. **Normalise with explicit loss masks.** Documents train on all tokens. Conversations train on everything except tool and environment responses, which in several sources arrive as user-role messages wrapped in `<tool_response>` tags, so a role check alone is not enough. Reasoning turns are kept.
3. **Deduplicate and decontaminate before sampling.** About 281 billion 13-word windows were scanned against a benchmark index of 5.2 million hashes and against held-out SFT data. 290 boilerplate 13-grams were reviewed and exempted; all other matches count as evidence for removal, and frequency alone never excludes a hash.
4. **Freeze the mixture by retained tokens.** After cleaning, 169.3M rows remain (about 529B tokens), of which 166.6M rows are eligible for training and 1.67M are held out for validation.

The mixture targets 30% reasoning/math/science, 25% code and agent data, 10% chat and instruction following, 12.5% long documents and 22.5% short-sequence replay, with roughly 65% raw documents and 35% conversations. Stage budgets are 0.5B, 1B and 2B tokens for 16K, 32K and 64K, a total of 3.5B tokens, with all 52 source weights fixed across stages and effective epochs kept under one.

The document also records where it deliberately reuses existing tooling and where that tooling would have corrupted the data: a generic text cleaner that strips `<think>` markers and collapses code indentation, hashing keys that lowercase code, a rebuild check that ignores tokenizer and mask changes, and tokenization with no conversation loss-mask stream.

## Context extension results

The quality gates were mean RULER of at least 85% and repeated-passkey needle retrieval of at least 95%, with 4K noninferiority against the parent model. The first recipe (RoPE theta 6M) failed badly. Changing theta alone lowered even 4K retrieval, which pointed to a recipe problem and not an infrastructure one. The revised recipe uses one theta of 9M for every stage, estimated from the critical dimension for 64K, then YaRN factor 2 for 128K.

| Checkpoint | RULER 4K | RULER 16K | RULER 32K | RULER 64K | RULER 128K |
|---|---:|---:|---:|---:|---:|
| Stage 1 (16K) | 87.4 | 74.1 | 60.2 | 12.6 | |
| Stage 2 (32K) | 88.0 | 78.7 | 71.2 | | |
| Stage 3 (64K), plain text | 83.9 | 80.5 | 75.0 | 56.3 | |
| Stage 4 (128K, YaRN x2), plain text | 85.0 | | | 65.8 | 50.4 |

Passkey retrieval reached 100% up to 64K from stage 2 on, and 99% at 128K after stage 4. The preregistered 85% RULER mean was not met at 16K, 32K or 64K. Failures were concentrated in multi-key retrieval, word counting and question answering. Part of the 4K drop at stage 3 was shown to be a format problem (answers ending immediately), not lost knowledge, and it largely disappears under a chat-style prompt.

Short-context few-shot results after stage 1 stayed within the preregistered margin of the parent on all six benchmarks, with large gains on MBPP+ and MultiPL-E. After stage 3, a paired bootstrap against the parent found no failures and no full pass: three metrics passed, and three (GSM8K, HumanEval+, MBPP+) were inconclusive. The author attributes the MBPP+ result to a stopping artifact and leaves the HumanEval+ drop unexplained.

Some decisions were made by the author to proceed despite failed gates, and are recorded as such in the source. Stage 4 is recommended as the base for the next SFT run, and chat and agent evaluations of it were still pending.

## SFT v2 data

A second document by the same author, [DATA.md](https://github.com/marianna13/prelude-posttrain/blob/main/sft/prelude/DATA.md), lists the data for the supervised fine-tuning that follows the context stages. SFT v2 starts from the stage 4 (128K) checkpoint and is trained in one pass at 64K context, 9,020 steps, on 37.7B seen tokens (30.1B trained). The same document also records the mid-training data summarised above, so the two stages can be read together.

**Sources**

| Source | Licence on the card |
|---|---|
| [PrimeIntellect/INTELLECT-3-SFT](https://huggingface.co/datasets/PrimeIntellect/INTELLECT-3-SFT) | Card empty; the upstream datasets need checking |
| [open-thoughts/AgentTrove](https://huggingface.co/datasets/open-thoughts/AgentTrove) | Apache-2.0 |
| [nebius/SWE-rebench-openhands-trajectories](https://huggingface.co/datasets/nebius/SWE-rebench-openhands-trajectories) | CC BY 4.0 |
| Instruction-following and chat rows from the author's earlier SFT v1 | See the source inventory |

SWE-bench Verified and a SWE reinforcement-learning pool are used only for overlap checks, and not for training.

**Mix**

| Part | Selection | Seen tokens |
|---|---|---:|
| Math reasoning (INTELLECT-3) | up to 4 traces per prompt, 634,354 rows | 11.97B |
| Code reasoning (INTELLECT-3) | up to 12 traces per prompt, 406,380 rows | 6.91B |
| Science reasoning (INTELLECT-3) | 53% sample, 847,922 rows | 5.48B |
| Chat and instruction following, thinking on | all, 1,005,318 rows | 2.37B |
| Tool use with native tool calls | all, 422,438 rows | 1.59B |
| SWE issue-to-patch | all, 9,934 rows | 0.17B |
| AgentTrove terminal-agent episodes | behaviour filter, longest episode per run, 595,509 rows | 7.29B |
| SWE-rebench OpenHands trajectories | resolved and ended with submit, 20,256 rows | 1.04B |
| Instruction following and chat, thinking off | reasoning stripped, 398,253 rows | 0.89B |
| **Total after rendering** | **4,340,196 rows** | **37.69B** |

**Processing**

- **Decontamination** against the evaluation suite (including SWE-bench Verified, Terminal-Bench 2.1 and TBLite) by 13-gram matching, with n-grams that appear in more than 20 distinct prompts treated as boilerplate. Instructions were also matched against several terminal-task sets. Most drops are small. The largest are SWE overlap in the SWE-rebench data (3,761 rows) and 11 TBLite tasks appearing verbatim in AgentTrove (2,012 rows). Exact prompt duplicates against earlier training data were flagged but not dropped.
- **Masking.** Only assistant turns train, tool and environment turns are masked, and reasoning is kept only after the last user turn. AgentTrove turns train only when they are valid structured commands or a handoff reply.
- **Rendering.** A 64K cap, with 842 over-long SWE-rebench rows and 168 unrenderable rows dropped. 570 rows with repetition loops or garbage tails (0.01%) were excluded at packing.

**Gaps the author lists:** deduplication is exact only, with no near-duplicate pass in either pipeline. The INTELLECT-3 licences are unresolved because the dataset card is empty. There is no inventory file for SFT v2 or for the stage 4 selection.

## Things worth reusing

- Count tokens and fix source weights by retained tokens after cleaning, and report both seen and trained shares.
- Record a decision for every excluded source and every rejected record.
- Fingerprint builds on the tokenizer, masks and benchmark index, and not only on input files.
- Keep failed gates as failed instead of lowering thresholds, and replay frozen evaluation inputs instead of regenerating them.
- Check long-context quality with several tasks. Passkey retrieval passed early while the mean score was still failing.

## Related

- [Validating RL environments](marianna13-rl-validation.md), also by Marianna13.
