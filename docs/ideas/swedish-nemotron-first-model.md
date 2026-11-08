# Training Nemotron 3.5 Lightning for Swedish

*[Svenska versionen](swedish-nemotron-first-model-sv.md)*

**Status:** proposal, open for comments
**Authors:** Renaissance Labs

We are considering [Nemotron 3.5 Lightning](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) (30B-A3B) as the first model we train for Swedish. This page sets out why, the training plan, and the questions we have not settled.

## Why this model

- **Architecture.** It is a hybrid of [Mamba-2](https://arxiv.org/abs/2405.21060) state-space layers, [mixture-of-experts](https://huggingface.co/blog/moe) layers and a few attention layers. It has 30B parameters in total with about 3B active per token, and a context of up to 1M tokens according to [NVIDIA's documentation](https://docs.api.nvidia.com/nim/reference/nvidia-nemotron-3-5-lightning-30b-a3b). Third-party hosts often default to a lower limit, such as 262K, so check the setting you actually run.
- **Size.** It is small enough to train and evaluate without a large allocation, and the quantised [NVFP4](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4) checkpoint makes local evaluation on a laptop realistic. It is still large enough to be useful, and cheap enough that a failed run is affordable.
- **Dogfooding.** Once the model is trained, we commit to using it for our own Swedish writing. A model we are forced to use is a model whose flaws we will find and fix.

## Training plan

1. **Supervised fine-tuning ([SFT](https://huggingface.co/docs/trl/sft_trainer))** on Swedish text, together with reasoning and tool-use data in both Swedish and English.
2. **Preference tuning with [DPO](https://arxiv.org/abs/2305.18290)** on Swedish values, so the model's answers reflect the norms we want it to hold. The draft [Svensk Konstitution](https://github.com/BirgerMoell/svensk-konstitution) (an open, unreviewed v0.1 text of values and judgement for Swedish-language AI agents, CC BY 4.0) can help in two ways: as the basis for generating preference pairs, where a teacher model answers, critiques and revises against the text and the student is then trained without the text in the prompt, and as the rubric for choosing the better answer.
3. **Reinforcement learning for good Swedish.** For each prompt the model produces several answers (around 8), a rubric-based judge model ranks them, and [GRPO](https://arxiv.org/abs/2402.03300) updates the model toward the best ones. Using an AI judge in place of human raters is called [RLAIF](https://arxiv.org/abs/2309.00267). The rubric can build on [Skrivbench](../projects/skrivbench.md). [Svenska RL](https://github.com/BirgerMoell/svenska-rl) adds verifiable Swedish environments (invoice amounts, payment matching, meeting scheduling, redaction and tool-use tasks, among others). They are scored by checking the result and not by a judge, so they complement the rubric judge. Eight environments exist today; the 20,000 in the repo is a planning target.

## Open questions

**Should the model reason in Swedish?** An earlier experiment got a Nemotron model to reason in Swedish. Reasoning in a language other than English usually lowers performance overall, but Swedish reasoning is still valuable to have. One option is to train two variants, one with Swedish reasoning and one without, and compare them. Our current view is that Swedish reasoning should be a requirement, and that we then have to recover the lost performance, which may be hard. We want other views on this.

**What data should we use?**
- Translated instruction and other datasets
- Open Swedish text, including government and parliament material (regering, riksdag and similar)
- Synthetic data generated with [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3), Codex and Claude

**How should we evaluate?** The evaluation has to reflect the goals. A first version can be simple: good Swedish, and no regression on other abilities.

## Data sources

Licences are as listed on each Hugging Face page. Where a page says "other" or lists none, check before using it, and note that synthetic data can also carry the terms of the model that generated it. Translated sets tend to carry anglicisms, which is what Skrivbench measures, so we want to mix them with original Swedish text.

**Pretraining and continued pretraining**

| Source | What it is | Licence |
|---|---|---|
| [FineWeb-2](https://huggingface.co/datasets/HuggingFaceFW/fineweb-2) (`swe_Latn`) | Swedish web text | ODC-By |
| [FinePDFs](https://huggingface.co/datasets/HuggingFaceFW/finepdfs) and [FinePDFs-Edu](https://huggingface.co/datasets/HuggingFaceFW/finepdfs-edu) (`swe_Latn`) | Long documents extracted from PDFs, well suited to long context | ODC-By |
| [FineWiki](https://huggingface.co/datasets/HuggingFaceFW/finewiki) (`sv`) | Cleaned Swedish Wikipedia | CC BY-SA 4.0, GFDL |
| [HPLT 2.0 cleaned](https://huggingface.co/datasets/HPLT/HPLT2.0_cleaned) (`swe_Latn`) | Web text | CC0 |
| [Swedish Dynaword](https://huggingface.co/datasets/danish-foundation-models/swedish-dynaword) | Curated open Swedish text, including government reports (SOU), historical laws and magazines | CC0 |
| [RixVox v2](https://huggingface.co/datasets/KBLab/rixvox-v2) | Riksdag speeches with transcripts (KBLab) | ODC-By |

For long-context work, [openeuro-longctx-datamix](https://github.com/BirgerMoell/openeuro-longctx-datamix) turns FinePDFs-Edu, Swedish included, into a weighted long-context data mix. [SWEb](https://arxiv.org/abs/2410.04456) is a large Scandinavian web corpus, but we could not confirm a public dataset page, so check access first.

**Supervised fine-tuning**

| Source | What it is | Licence |
|---|---|---|
| [Dolci-Instruct-SFT-translated](https://huggingface.co/datasets/openeurollm/Dolci-Instruct-SFT-translated) (`sv`) | Instruction data translated by OpenEuroLLM | Apache-2.0 |
| [Dolci-Think-SFT-translated](https://huggingface.co/datasets/openeurollm/Dolci-Think-SFT-translated) (`sv`) | Reasoning data, translated | Apache-2.0 |
| [EuroMultiSynth-Instruct](https://huggingface.co/datasets/openeurollm/EuroMultiSynth-Instruct) (`sv`) | Multilingual synthetic instruction data | Apache-2.0 |
| [smol-smoltalk-swedish](https://huggingface.co/datasets/liu-nlp/smol-smoltalk-swedish) | About 457,000 Swedish conversations (Linköping University) | Not stated |
| [reasoning-traces-multilingual](https://huggingface.co/datasets/openeurollm/reasoning-traces-multilingual) | Multilingual reasoning traces, Swedish included | CC BY 4.0 |
| [openeurollm-reasoning-instruct-v1](https://huggingface.co/datasets/birgermoell/openeurollm-reasoning-instruct-v1) and [openeurollm-tool-use-v1](https://huggingface.co/datasets/birgermoell/openeurollm-tool-use-v1) | Reasoning and tool-use data | Other |

**Preference data (DPO)**

| Source | What it is | Licence |
|---|---|---|
| [Dolci-Instruct-DPO-translated](https://huggingface.co/datasets/openeurollm/Dolci-Instruct-DPO-translated) | Preference pairs translated by OpenEuroLLM | Apache-2.0 |
| [dpo-swedish-interpreted](https://huggingface.co/datasets/jonasaise/dpo-swedish-interpreted) | UltraFeedback pairs adapted to Swedish with Kimi K3. The rejected answers are poor translations, so it teaches natural phrasing more than values. | MIT |
| [oellm-dpo-metadataset](https://huggingface.co/datasets/birgermoell/oellm-dpo-metadataset) | DPO collection with Swedish included. We have not yet inspected the contents. | Apache-2.0 |

**Reinforcement learning and prompts**

- [Svenska RL](https://github.com/BirgerMoell/svenska-rl): verifiable Swedish environments, with an export to RL training data.
- [oellm-math-rlvr](https://huggingface.co/datasets/openeurollm/oellm-math-rlvr): math problems with checkable answers (Apache-2.0).
- [swedish-personas](https://huggingface.co/datasets/birgermoell/swedish-personas): 100,000 synthetic Swedish personas sampled from SCB demographics, useful for varying prompts (CC BY 4.0).

**Catalogues and related work.** The [OpenEuroLLM post-training data registry](https://huggingface.co/datasets/openeurollm/post-training-data-registry), the [post-training-data](https://github.com/BirgerMoell/post-training-data) catalogue and the [training-data-catalogue](https://github.com/OpenEuroLLM/training-data-catalogue) list what exists across languages. [nemotron-swedish-posttraining](https://github.com/BirgerMoell/nemotron-swedish-posttraining) holds earlier Swedish post-training experiments for Nemotron, and [svea-eval](https://github.com/BirgerMoell/svea-eval) and [svbench-v2](https://github.com/BirgerMoell/svbench-v2) are Swedish evaluations we can use for the no-regression check. More places to look: [birgermoell on Hugging Face](https://huggingface.co/birgermoell), [openeurollm on Hugging Face](https://huggingface.co/openeurollm), [BirgerMoell on GitHub](https://github.com/BirgerMoell) and [OpenEuroLLM on GitHub](https://github.com/OpenEuroLLM).

## How to run the experiments

- **Start small.** Begin with a [LoRA](https://arxiv.org/abs/2106.09685) adapter, or a full training run if the cost turns out to be acceptable.
- **Budget before every run.** Ask what a run costs in GPU hours (a coding agent can estimate this), what capability it trains and what we expect to learn from it. Small experiments come first.
- **Then plan the big run.** Once the small runs have shown what works, structure the data, and make one person responsible for the training itself.
- **Do run things on the cluster.** Hesitating to start runs is the more costly mistake. Keep an eye on the budget and the number of GPUs, and otherwise go ahead. The [OpenEuroLLM](https://openeurollm.eu) project, which trains open European language models, is a useful reference for how large multilingual efforts organise this work.

## Links

- Model: [Hugging Face (BF16)](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16), [NVIDIA NIM documentation](https://docs.api.nvidia.com/nim/reference/nvidia-nemotron-3-5-lightning-30b-a3b)
- Mamba: [Mamba](https://arxiv.org/abs/2312.00752) and [Mamba-2](https://arxiv.org/abs/2405.21060)
- Methods: [LoRA](https://arxiv.org/abs/2106.09685), [DPO](https://arxiv.org/abs/2305.18290), [GRPO](https://arxiv.org/abs/2402.03300), [RLAIF](https://arxiv.org/abs/2309.00267)
- Data generator: [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3)
- Project reference: [OpenEuroLLM](https://openeurollm.eu)
