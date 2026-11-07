# How a frontier model is trained

**Status:** finished (notes behind a film)
**Authors:** Renaissance Labs

A researched overview of what it takes to train a frontier language model in 2026: data, scaling, chips, parallelism, failures, mixture-of-experts, post-training and cost. We deliberately used only 2026 sources, because older ones (Llama 3, Chinchilla, DeepSeek-V3) no longer describe the frontier.

## The shape of the recipe

1. **Data.** Base corpora are now in the tens of trillions of tokens (GLM-5: 27T for the base corpus, 28.5T across all stages; DeepSeek-V4: over 32T). Quality filtering matters as much as volume: classifiers for educational and world-knowledge content, fuzzy deduplication, and removing templated web text. Agentic coding data is added mid-training.
2. **Scale.** The big open models are mixture-of-experts. GLM-5 has 744B parameters in total and 40B active. DeepSeek-V4 Pro has 1.6T total and 49B active. Kimi K2.5 has 1.04T total and 32B active, with 384 experts and 8 active per token.
3. **Context.** Context is extended in stages (GLM-5 goes 32K, then 128K, then 200K). DeepSeek-V4 reaches 1M tokens, using 27% of the FLOPs and 10% of the KV cache of its predecessor at that length.
4. **Hardware and parallelism.** Training combines pipeline, expert and data parallelism. Kimi K2.5 reports 16-way pipeline and 16-way expert parallelism on H800 GPUs with an 8×400G network. GLM-5 reports running on seven domestic chip platforms.
5. **Optimisers.** All three models use a Muon-family optimiser.
6. **Post-training.** This is now a large share of the work: reinforcement learning on tens of thousands of verifiable environments (GLM-5 reports over 10,000 software-engineering ones), asynchronous agentic RL, and multi-teacher on-policy distillation. DeepSeek-V4 runs hundreds of thousands of concurrent sandboxes for this. Reward hacking is a constant problem; in one case the fix was to repair the renderer being exploited, not to punish the model.

## Failures are normal at scale

Meta's 2026 paper on fault-tolerant training at 100,000 GPUs reports, for a 32K-GPU job, about 2.3 interruptions per 1,000 servers per day, 78% of them hardware, with 95–97% effective training time. At 100K GPUs a failure arrives roughly every 18 minutes, and the paper's own model puts effective time near 44% if recovery is fully synchronous. Silent data corruption (a chip returning a wrong answer without an error) is an active research area.

## Cost

None of the three model reports gives GPU hours or a dollar figure. What exists comes from Epoch AI's 2026 infrastructure analysis: a 1 GW AI data centre costs about $38B to build and about $8.5B a year in total cost of ownership, with servers around 60% of it. xAI's Colossus 2 was about 950 MW in September 2026. Epoch estimates AI-related investment at about 0.8% of US GDP in Q1 2026.

## Numbers we derived

Training compute is about 6 × active parameters × tokens. That gives roughly 7e24 FLOPs for GLM-5 and 1e25 for DeepSeek-V4 Pro. These are our arithmetic, not reported values.

## Caveats

- Several details come from model-written summaries of the papers and not from the PDFs, so they are medium confidence. Recheck before relying on them.
- A social-media post described "six data engines" for a later GLM model. We could not confirm it anywhere and left it out.

## Sources

- GLM-5, Z.ai, February 2026, [arXiv 2602.15763](https://arxiv.org/abs/2602.15763)
- DeepSeek-V4, [arXiv 2606.19348](https://arxiv.org/abs/2606.19348)
- Kimi K2.5, [arXiv 2602.02276](https://arxiv.org/abs/2602.02276)
- Salpekar et al., *Training LLMs with Fault Tolerant HSDP on 100,000 GPUs*, [arXiv 2602.00277](https://arxiv.org/abs/2602.00277)
- Altenbernd, Wiesner, Kao, *Exploring Silent Data Corruption as a Reliability Challenge in LLM Training*, [arXiv 2604.00726](https://arxiv.org/abs/2604.00726)
- Epoch AI data insights: frontier data centre power, largest data centre compute, AI data centre cost breakdown, AI share of US GDP (all 2026), at [epoch.ai](https://epoch.ai/data-insights)
