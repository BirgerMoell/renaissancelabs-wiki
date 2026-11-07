# GPU hardware limits: what is silicon and what is software

**Status:** finished (notes behind a film)
**Authors:** Renaissance Labs

Hardware and software are easy to mix up when talking about low-precision AI. This page separates what a GPU's tensor cores can do in silicon from what a software kernel can arrange. Research date: 7 November 2026. Every claim was read from a primary source where we could.

## The datasheet trap

NVIDIA's headline tensor numbers are **sparse** (2:4 structured sparsity), which doubles them. The numbers below are **dense**.

## Dense peak rates

| GPU | Format | Dense peak | Memory |
|---|---|---|---|
| H100 SXM | BF16 / FP16 | 989.5 TFLOPS | 80 GB HBM3, 3.35 TB/s |
| H100 SXM | FP8 | 1,979 TFLOPS | |
| H100 SXM | FP32 (CUDA cores) | 67 TFLOPS | |
| B200 (per GPU) | BF16 / FP8 / FP4 | 2.5 / 5 / 10 PFLOPS | up to 192 GB HBM3e, 8 TB/s |
| B300 (per GPU) | FP4 | 15 PFLOPS (BF16, FP8 unchanged from B200) | up to 288 GB HBM3e, 8 TB/s |
| MI300X | FP16 / FP8 | 1,307 / 2,615 TFLOPS | 192 GB HBM3, 5.3 TB/s |
| MI355X | FP16 / FP8 / FP4 and FP6 | 2,517 / 5,033 / 10,066 TFLOPS | 288 GB HBM3E, 8 TB/s |

Two ratios worth knowing: on H100 the tensor cores run BF16 about 15× and FP8 about 30× faster than the FP32 CUDA cores. And decoding a model at batch size 1 does about 1 FLOP per byte of weights in BF16 (about 4 in 4-bit), far below the roughly 300 FLOP per byte that H100 needs to be compute-bound, so decode is limited by memory bandwidth.

## Hopper has no 4-bit tensor-core path

The Hopper tensor-core instruction (`wgmma`) supports f16, bf16, tf32, e4m3/e5m2, s8/u8 and b1. It has no INT4, FP4 or FP6 types, and its scale operands only negate. Native FP4 and block scaling arrive with Blackwell's `tcgen05.mma` (scales read from Tensor Memory) and, on AMD, with CDNA4's `v_mfma_scale` instructions on the MI355X. So 4-bit on Hopper is a software story: Marlin-style kernels keep 4-bit weights, dequantise them, and multiply in FP16. This helps a lot for small batches (Marlin reports near-ideal speedup up to batch 16–32, about 2.8× end to end in vLLM) and little for compute-heavy work.

## NVFP4 and block scales

NVFP4 stores values as E2M1 (magnitudes 0, 0.5, 1, 1.5, 2, 3, 4, 6), with one E4M3 scale per block of 16 values and an FP32 scale per tensor. MXFP4 uses blocks of 32 with a power-of-two scale. NVIDIA's training recipe keeps the first two and last eight blocks in BF16, uses random Hadamard transforms on weight-gradient inputs, and rounds gradients stochastically. On a 12B model trained on 10T tokens it matched FP8 on MMLU-Pro (62.58 vs 62.62), while MXFP4 needed 36% more tokens.

We also checked a small question ourselves: how do products of an FP4 value and an E4M3 scale fit in a wider format? Of 1,016 distinct products, all are exact in BF16, but only 741 (72.9%) are exact in E4M3. This is our own calculation. It shows why block scales are cheap in hardware, but it is not a claim that Hopper can run them natively. Hopper can do FP8 matrix multiplies with scaling applied on the CUDA cores, as DeepSeek does, which is a software scheme with extra work.

## Smaller facts

- AMD's MI300 uses FNUZ FP8 (max 240), while CDNA4 moves to OCP E4M3 (max 448).
- DeepSeek-V3 measured that H800 FP8 accumulation keeps about 14 bits, which is why they promote partial sums to FP32 every 128 products. This is a third-party measurement, not an NVIDIA specification.

## Not verified, left out

Hopper's per-partition warp scheduler, the B200 SM count, an "MI355X 20 PF" figure that appears only in press, and any explanation for why Blackwell Ultra cut FP64 throughput.

## Sources

- NVIDIA: [H100](https://www.nvidia.com/en-us/data-center/h100/), [GB200 NVL72](https://www.nvidia.com/en-us/data-center/gb200-nvl72/), [GB300 NVL72](https://www.nvidia.com/en-us/data-center/gb300-nvl72/), [Hopper architecture in depth](https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/), [PTX ISA](https://docs.nvidia.com/cuda/parallel-thread-execution/index.html)
- AMD: Instinct MI300X and MI355X datasheets, CDNA3 and CDNA4 whitepapers and ISA documents at [amd.com](https://www.amd.com/en/products/accelerators/instinct.html)
- [NVFP4 pretraining paper](https://arxiv.org/abs/2509.25149) and [NVIDIA's NVFP4 introduction](https://developer.nvidia.com/blog/introducing-nvfp4-for-efficient-and-accurate-low-precision-inference/)
- [FP8 formats for deep learning](https://arxiv.org/abs/2209.05433)
- [Marlin](https://arxiv.org/abs/2408.11743) and [DeepSeek-V3](https://arxiv.org/abs/2412.19437)
- [Roofline model](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2008/EECS-2008-134.pdf)
