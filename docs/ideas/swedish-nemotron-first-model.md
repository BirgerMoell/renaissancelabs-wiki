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
2. **Preference tuning with [DPO](https://arxiv.org/abs/2305.18290)** on Swedish values, so the model's answers reflect the norms we want it to hold.
3. **Reinforcement learning for good Swedish.** For each prompt the model produces several answers (around 8), a rubric-based judge model ranks them, and [GRPO](https://arxiv.org/abs/2402.03300) updates the model toward the best ones. Using an AI judge in place of human raters is called [RLAIF](https://arxiv.org/abs/2309.00267). The rubric can build on [Skrivbench](../projects/skrivbench.md).

## Open questions

**Should the model reason in Swedish?** An earlier experiment got a Nemotron model to reason in Swedish. Reasoning in a language other than English usually lowers performance overall, but Swedish reasoning is still valuable to have. One option is to train two variants, one with Swedish reasoning and one without, and compare them. Our current view is that Swedish reasoning should be a requirement, and that we then have to recover the lost performance, which may be hard. We want other views on this.

**What data should we use?**
- Translated instruction and other datasets
- Open Swedish text, including government and parliament material (regering, riksdag and similar)
- Synthetic data generated with [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3), Codex and Claude

**How should we evaluate?** The evaluation has to reflect the goals. A first version can be simple: good Swedish, and no regression on other abilities.

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
